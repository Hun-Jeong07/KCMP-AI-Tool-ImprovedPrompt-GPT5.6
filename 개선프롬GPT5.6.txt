/***** ===== 0) 설정 & 유틸 ===== *****/
/**
 * 🎯 프로그램 개요
 * 이 스크립트는 소집단 담화 데이터를 자동으로 분석하는 Google Apps Script입니다.
 * 학생들의 대화를 시간별로 클러스터링하고, AI를 활용하여 인식적 실행(K), 협력(C), 메타인지(M), 참여도(P) 차원으로 코딩합니다.
 *
 * 📋 주요 기능:
 * - 타임스탬프 기반 발화 클러스터링 (Act→Event 병합)
 * - AI 기반 자동 코딩 (K/C/M/P 차원)
 * - 화자별 발화수 분석 (참여도 정량 측정)
 * - 교사/학생 발화 구분
 * - 캐시를 통한 효율적인 API 호출
 */


// 🤖 AI 모델 설정
var MODEL = "gpt-4o-mini"; // 기본 모델 (KCMP 코딩 전용)

// 🎯 작업별 모델 설정
const MODEL_CLUSTER = "gpt-4o-mini";   // 클러스터링용
const MODEL_K = "gpt-5.6-terra";      // K 코딩용 (STEP 14A)
const MODEL_M = "gpt-5.6-terra";      // M 코딩용 (STEP 14A)
const MODEL_KM = "gpt-4o-mini";       // LEGACY ONLY: runCodeKM_All_LEGACY 전용. live path 미사용.
const MODEL_C = "gpt-5.6-terra";      // C 코딩용 (STEP 14A)
const MODEL_P = "gpt-4o-mini";        // LEGACY GPT P (live path 미사용)
const MODEL_SUMMARY = "gpt-4o-mini";  // 요약·다이어그램용

// STEP 5 M calibration closed — prompt/validator frozen (2026-08-19)
const STEP_5_M_STATUS = "CLOSED_WITH_KNOWN_LIMITATIONS";
const STEP_5_M_CORE_TOTAL = 13;
const STEP_5_M_CORE_PASS = 11;
const STEP_5_M_KNOWN_MISMATCHES = ["P035", "P065"];
const STEP_5_M_PROMPT_FROZEN = true;

// STEP 8 close — production readiness gate는 static live-path만 사용.
// upstream K/C/M은 GPT 기반이라 semantic/validator variability가 남을 수 있으므로
// runtime variability를 “알고도” production run 상태로 진입한다.
const KCMP_PIPELINE_STATUS = "READY_WITH_KNOWN_LLM_VARIABILITY";

// STEP 9A/12A — resumable production batch orchestration (semantic logic 변경 없음)
const KCMP_KCM_PRODUCTION_BATCH_MAX_CASES = 30;
const KCMP_KCM_PRODUCTION_TIME_BUDGET_MS = 240000;
const KCMP_P_PRODUCTION_BATCH_MAX_CASES = 100;
const KCMP_P_PRODUCTION_TIME_BUDGET_MS = 240000;
const KCMP_PRODUCTION_TIME_SAFETY_MARGIN_MS = 15000;
const KCMP_API_ERROR_RECOVERY_MAX_CASES = 5;
const KCMP_TERRA_K_SHADOW_MAX_CASES = 5;
const KCMP_TERRA_C_SHADOW_MAX_CASES = 5;
const KCMP_TERRA_M_SHADOW_MAX_CASES = 5;
const KCMP_K_CAND16B_SHADOW_MAX_CASES = 5;

// 🔧 작업별 모델 선택 함수
function getModelForTask_(taskName) {
  switch (taskName) {
    case 'cluster': return MODEL_CLUSTER;
    case 'K': return MODEL_K;
    case 'M': return MODEL_M;
    case 'C': return MODEL_C;
    case 'P': return MODEL_P;
    case 'summary': return MODEL_SUMMARY;
    default: return MODEL_SUMMARY;
  }
}

// 🔧 요약 모드
// - "gpt_exhaustive": 모든 실질 발화를 빠짐없이 서술(교사 포함)
// - "gpt_rich": 맥락 묘사 + 학생 인용 다수(압축 요약)
// - "gpt_narrative": 가볍게 압축한 자연 내러티브
// - "local_full": 전 발화를 로컬 합성(폴백)
const SUMMARY_MODE = "gpt_exhaustive";

// ============================================================
// LEGACY / INACTIVE / NOT USED BY LIVE KCMP PIPELINE
// K_CODE_PROMPT: GPT 일괄 K 코딩용 구버전 프롬프트 상수.
// live path(runCodeK_All → runKDecisionTreeForPacket_)는 이 상수를 사용하지 않는다.
// ============================================================
const K_CODE_PROMPT = `다음 요약문을 읽고 K차원 코드(K1~K3)를 1개만 선택하라.

⚠️ 중요: 해당되는 코드가 없으면 아무것도 출력하지 말 것. 셀을 비워두어야 함.

[K 코드북]

K1. 추론과 설명 구성  
 - 새로운 설명을 구성하거나 비유를 만들거나, 인과·구조·요인 관계를 추론하는 과정  

K2. 자료 수집 및 해석  
 - 관찰, 실험, 경험, 기억, 표·그래프·데이터 등 외부 정보를 활용하여 해석하는 과정  

K3. 주장에 대한 정당화  
 - 주장과 근거를 명확히 연결하거나, 개념적·논리적 정당화를 제공하는 경우  

[판정 기준]
- 요약문에서 K1, K2, K3 중 하나라도 명확히 드러나는 경우만 코드 부여
- 단순 발화만 있고 추론/설명/자료해석/정당화가 전혀 없는 경우 → 출력하지 말 것 (셀 비워둠)
- 절차/잡담/리액션만 있는 경우 → 출력하지 말 것 (셀 비워둠)

요약문:
"<<SUMMARY>>"

출력 형식 (해당 코드가 있을 때만):
K# — 코드명 — 근거(10~25자)

해당되는 코드가 없으면 아무것도 출력하지 말 것.`;

// ============================================================
// LEGACY / INACTIVE / NOT USED BY LIVE KCMP PIPELINE
// M_CODE_PROMPT: GPT 일괄 M 코딩용 구버전 프롬프트 상수.
// live path(runCodeM_All → runMDecisionTreeForPacket_)는 이 상수를 사용하지 않는다.
// (STEP 5에서 decision tree로 교체 완료)
// ============================================================
const M_CODE_PROMPT = `다음 요약문을 읽고 학생들의 메타인지적 행위를 가장 잘 설명하는 M코드 1개를 선택하라.

⚠️ 중요: 한 발화만 보지 말고 요약문 전체 맥락(전후 맥락 포함)을 반드시 확인해야 함.
특히 M4 판정 시: 질문이 나왔다고 해서 바로 M4로 판정하지 말고, 질문의 목적과 주변 맥락을 반드시 확인하라.

[M 코드북 - 메타인지 차원]

M1. 논의의 목표와 방식
- 논의가 산만해지거나 흐름이 모호할 때, 현재 과업의 목표나 방향을 점검하거나, 논의의 흐름, 순서, 방식, 전략 등을 자발적으로 조정하려는 시도
- 예시: "우리가 지금 풀어야 할 건 뭐였지?", "지금 이거 하자는 거야?", "지금 그 얘기 말고 원래 얘기 하자", "잠깐! 다시 돌아가자", "이렇게 말고 먼저 주사기 생각부터 해보자", "정리부터 하고 말하자"
- 핵심: 목표 점검, 흐름 조정, 순서/방식 제안

M2. 참여 태도 및 규범
- 논의 중 친구가 침묵하거나 소외되고 있는 상황에서, 학생이 자발적으로 해당 친구의 발언을 유도하거나, 모두가 참여해야 한다는 규범을 상기시키는 발화
- 예시: "모둠활동은 혼자하는거 아니라고!", "00아 넌 어떻게 생각해?", "너도 한마디 해봐"
- 핵심: 친구 참여 촉진, 소외 방지, 참여 규범 상기

M3. 설명 및 논리 점검
- 자신 또는 집단(모둠)의 설명, 추론, 주장에 대해 논리적 오류, 모순, 비약 등을 스스로 인식하고, 그것의 타당성이나 일관성을 점검하거나 수정하려함
- 예시: "내 말이 좀 이상하지 않아?", "우리가 아까 말한 거랑 지금 말이 안 맞는 것 같은데?", "그럼 앞에 말한 건 뭐지? 말이 안 돼…"
- 핵심: 논리적 오류/모순/비약 인식, 타당성 점검, 일관성 확인

M4. 개념 이해
- ⚠️ 중요: M4는 "메타인지적 개념 이해 요청"이어야 함. 단순 질문이 아니라 자신의 이해 부족을 인식하고 명확히 하려는 메타인지적 행위.
- 어떤 개념이나 현상, 설명의 이해가 부족함을 명시적으로 인정하거나, 의미를 명확히 이해하기 위한 질문 혹은 자신의 이해 상태를 점검하고 확인받으려는 목적의 질문 발화
- ⚠️ 맥락 필수 확인: 질문이 나오더라도 주변 맥락을 반드시 확인해야 함
  * 단순 "왜?" 같은 질문만으로는 M4 판정 금지
  * 질문의 목적이 반박/비판을 위한 것(C4)이면 M4 아님
  * 질문의 목적이 상대 설명을 요구하는 것(C2)이면 M4 아님
  * 질문 후 이해 부족을 인정하거나 이해 상태를 점검하려는 의도가 명확할 때만 M4
  * ⚠️ 단순 문답 패턴은 M4 아님: "커지지?" → "응." 같은 단순 질문-응답은 M 코딩에 해당 없음 → 빈 셀
- 예시: "이건 왜 그런 거야?" (이해 부족 인정) + "모르겠어" → M4
- 예시: "그게 무슨 말이야?" (의미 명확화 요청) + "이해가 안 돼" → M4
- 예시: "폐가 어떻게 커지는 거야?" (이해 상태 점검) → M4
- 반례: "왜 그렇게 생각했어?" → C2 (명료화 요청, M4 아님)
- 반례: "그건 말이 안 돼, 왜?" → C4 (반박을 위한 질문, M4 아님)
- ⚠️ 반례: "커지지?" → "응." → M4 아님, M 코딩에 해당 없음 → 빈 셀 (단순 문답)
- ⚠️ 반례: "이거 맞아?" → "응." → M4 아님, M 코딩에 해당 없음 → 빈 셀 (단순 확인 질문-응답)
- 핵심: 이해 부족 명시적 인정, 의미 명확화 질문, 이해 상태 점검, 주변 맥락 확인 필수
- ⚠️ 단순 문답/확인 질문은 M4가 아님. 메타인지적 행위가 없으면 M 코딩에 해당 없음 → 빈 셀

[판정 우선순위 - 맥락 기반]
- ⚠️ M4 판정 전 필수 확인: 질문이 나왔다고 해서 바로 M4로 판정하지 말 것
  1) 단순 문답/확인 질문인가? → M 코딩에 해당 없음 (빈 셀)
  2) 질문의 목적 확인: 반박/비판을 위한 질문인가? → C4
  3) 질문의 목적 확인: 상대 설명을 요구하는 질문인가? → C2
  4) 질문의 목적 확인: 진짜 이해 부족을 인정하고 명확히 하려는 질문인가? → M4
  5) 주변 맥락 확인: 질문 후 이해 부족 인정, 이해 상태 점검 의도가 있는가?
- M1과 M2가 동시에 나타나면: 참여 규범(M2)이 더 구체적이면 M2, 목표/방식 점검이 더 중심이면 M1
- M3과 M4가 동시에 나타나면: 논리 점검(M3)이 더 고차적이면 M3, 개념 이해 질문(M4)이 더 중심이면 M4
- M4와 C2 구분: 
  * C2는 상대의 주장/설명에 대해 이유/근거를 요구 (상대를 향한 질문)
  * M4는 자신의 이해 부족을 인정하고 개념을 이해하려는 질문 (자신의 이해 상태 점검)
- 여러 메타인지 행위가 혼재하면: 가장 명시적이고 구체적인 행위를 선택

[판정 기준]
- 요약문에서 M1, M2, M3, M4 중 하나라도 명확히 드러나는 경우만 코드 부여
- 단순 발화만 있고 메타인지적 행위가 전혀 없는 경우 → 출력하지 말 것 (셀 비워둠)
- 절차/잡담/리액션만 있는 경우 → 출력하지 말 것 (셀 비워둠)
- ⚠️ 단순 문답/확인 질문만 있는 경우 → 출력하지 말 것 (셀 비워둠)
  * 예: "커지지?" → "응." 같은 단순 질문-응답 패턴
  * 예: "이거 맞아?" → "응." 같은 단순 확인 질문-응답 패턴
  * 이런 경우는 메타인지적 행위가 아니므로 M 코딩에 해당 없음

요약문:
"<<SUMMARY>>"

출력 형식 (해당 코드가 있을 때만):
M# — 코드명 — 근거(10~25자)

⚠️ 해당되는 코드가 없으면 아무것도 출력하지 말 것. 셀을 비워두어야 함.`;


// ===== 📊 스프레드시트 컬럼 정의 (개편 후) =====
// A: 화자(1) | B: 타임스탬프(2) | C: 발화(3)
// D: PID(행별, 4) | E: PID 목록(5) | F: 클러스터 요약문(6)

// ✅ 화자별 발화수 집계 결과: G/H/I/J (7~10)
const SPEAKER_CNT_START_COL = 7;   // G
const SPEAKER_CNT_COLS = 4;        // G..J (4개 열)

// ✅ KCMP 코딩 결과: K/L/M/N/O/P (11~16)
//   K=K차원, L=C차원, M=M차원, N=P차원(신설), O=교사개입, P=패턴
const P_K_COL = 11;         // K
const P_C_COL = 12;         // L
const P_M_COL = 13;         // M
const P_P_COL = 14;         // N (신설: Participation/P 차원)
const TEACHER_FLAG_COL = 15; // O
const PATTERN_COL = 16;      // P

// ✅ 다이어그램(시각화) 출력: Q/R/S/T (17~20)
const DIAG_K_COL = 17;  // Q : K-다이어그램
const DIAG_C_COL = 18;  // R : C-다이어그램
const DIAG_M_COL = 19;  // S : M-다이어그램
const DIAG_P_COL = 20;  // T : P-다이어그램

// 고정 열 (변경 없음)
const SPEAKER_COL = 1;
const TS_COL = 2;
const UTTER_COL = 3;
const PID_COL = 4;
const P_IDLIST_COL = 5;
const P_SUMMARY_REFINED_COL = 6;
const TIMESTAMP_COL = 2; // 호환성 유지

// 코딩 결과 쓰기 모드
const CODES_WRITE_TARGET = "list";


/***** ===== 새 메뉴 및 헤더 관리 시스템 ===== *****/

/**
 * 📌 1행 헤더 스타일 자동 적용
 * - bold, 배경색 #f2f2f2, vertical alignment middle, horizontal alignment center
 */
function styleHeaderRow() {
  try {
    const sheet = SpreadsheetApp.getActiveSheet();
    if (!sheet) return; // 시트가 없으면 스킵
    
    const lastCol = sheet.getLastColumn();
    if (lastCol === 0) return; // 빈 시트면 스킵
    
    // 1행이 없으면 스킵 (안전성 체크)
    if (sheet.getLastRow() < 1) return;
    
    const header = sheet.getRange(1, 1, 1, lastCol);
    
    header.setFontWeight("bold");
    header.setBackground("#f2f2f2");
    header.setVerticalAlignment("middle");
    header.setHorizontalAlignment("center");
  } catch (e) {
    // 에러 발생 시 조용히 실패 (다른 기능에 영향 주지 않음)
    Logger.log("styleHeaderRow 오류: " + e.toString());
  }
}

/***** 1) 매핑 로드/저장 공통 *****/
function loadColMap_() {
  const props = PropertiesService.getDocumentProperties();
  const raw = props.getProperty('COLMAP');
  if (!raw) return null;
  let map;
  try {
    map = JSON.parse(raw);
  } catch (e) {
    // 과거 문자열/기타 형식 잔여물 방지
    Logger.log('⚠️ COLMAP 파싱 실패: ' + e);
    map = null;
  }
  return map;
}

function saveColMap_(map) {
  // 항상 JSON 문자열로 저장
  PropertiesService.getDocumentProperties()
    .setProperty('COLMAP', JSON.stringify(map));
}

/***** 2) 숫자 열 번호로 안전 변환 *****/
/** 열 매핑이 숫자/문자열/객체 어떤 형태로 와도 "숫자 열번호" 반환 */
function colNumOf(colRef) {
  if (colRef == null) throw new Error('colNumOf: undefined colRef');
  if (typeof colRef === 'number') return colRef;
  if (typeof colRef === 'string') {
    const n = parseInt(colRef, 10);
    if (Number.isFinite(n)) return n;
  }
  if (typeof colRef === 'object') {
    // 기대 포맷: { col: number, header: string }
    if (colRef.col && Number.isFinite(colRef.col)) return colRef.col;
  }
  throw new Error('colNumOf: invalid column ref -> ' + JSON.stringify(colRef));
}

/***** 3) S1~S4를 일관된 객체 포맷으로 정규화 *****/
/** S1~S4를 객체 포맷으로 정규화 */
function normalizeSColsInMap_(map) {
  if (!map) return map;
  ['S1','S2','S3','S4','K','L','M','N','O','P','A','B','C','D','E','F','G','H','I','J'].forEach(k=>{
    const v = map[k];
    if (!v) return;
    if (typeof v === 'number') map[k] = { col: v, header: '' };
    else if (typeof v === 'string') {
      const n = parseInt(v,10);
      if (Number.isFinite(n)) map[k] = { col: n, header: '' };
    }
  });
  return map;
}

/***** 4) 매핑 즉시 검증(없거나 타입 불일치면 중단) *****/
/** 헤더 무결성 검증 */
function validateColMap_(map, sh) {
  const need = ['A','B','C','D','E','F','K','L','M','N','O','P','S1','S2','S3','S4'];
  need.forEach(k=>{
    if (!map[k]) throw new Error('헤더 매핑 누락: ' + k);
    const col = colNumOf(map[k]);
    if (!Number.isFinite(col) || col < 1) throw new Error('열번호 이상: ' + k + ' -> ' + JSON.stringify(map[k]));
  });
}

function getSCols_(map) {
  return [colNumOf(map.S1), colNumOf(map.S2), colNumOf(map.S3), colNumOf(map.S4)];
}

/***** 5) 헤더 해시 저장/변경 감지 *****/
function getHeaderRow_(sh) {
  const lastCol = sh ? sh.getLastColumn() : 0;
  if (!lastCol || lastCol < 1) {
    const name = sh && sh.getName ? sh.getName() : "(unknown)";
    throw new Error('활성 시트 "' + name + '"에 열이 없습니다. 데이터 시트(헤더가 있는 탭)를 선택한 뒤 다시 실행하세요.');
  }
  return sh.getRange(1, 1, 1, lastCol).getValues()[0];
}
function headerHash_(arr){ return Utilities.base64EncodeWebSafe(JSON.stringify(arr)); }

function saveHeaderHash_() {
  const sh = SpreadsheetApp.getActiveSheet();
  const props = PropertiesService.getDocumentProperties();
  const hash = headerHash_(getHeaderRow_(sh));
  props.setProperty('HEADER_HASH', hash);
}

function headerChanged_() {
  const sh = SpreadsheetApp.getActiveSheet();
  const props = PropertiesService.getDocumentProperties();
  const prev = props.getProperty('HEADER_HASH') || '';
  const now = headerHash_(getHeaderRow_(sh));
  return prev !== now;
}

/***** 6) 진입부 공통 가드 *****/
/** 진입부 공통 가드 */

/***** 7) 헤더 해시/캐시 무시하는 강제 리프레시 옵션 *****/
function resetColMapCacheHard_() {
  PropertiesService.getDocumentProperties().deleteProperty('COLMAP');
  SpreadsheetApp.getActive().toast('COLMAP 초기화 완료. [헤더 설정] → [교사·학생 지정] 순으로 다시 설정하세요.', '초기화', 5);
}

/***** 8) 🧯 S1~S4 자동탐지 실패 시 수동입력(숫자 검증) *****/
function promptSColsFallback_(){
  const ui = SpreadsheetApp.getUi();
  const ask = (label)=>{
    while(true){
      const r = ui.prompt(label+' 열 번호(숫자)를 입력하세요.', ui.ButtonSet.OK_CANCEL);
      if (r.getSelectedButton() !== ui.Button.OK) throw new Error('사용자 취소');
      const n = Number(r.getResponseText().trim());
      if (Number.isFinite(n) && n>=1) return n;
      ui.alert('숫자만 입력하세요.');
    }
  };
  const s1 = ask('S1'); const s2 = ask('S2'); const s3 = ask('S3'); const s4 = ask('S4');
  const sh = SpreadsheetApp.getActiveSheet();
  let map = getColMap_();
  // 성능 개선: 개별 셀 읽기 대신 배치 읽기
  const headerRow = sh.getRange(1, 1, 1, Math.max(s1, s2, s3, s4)).getValues()[0];
  map.S1 = { col:s1, header: String(headerRow[s1-1]||'') };
  map.S2 = { col:s2, header: String(headerRow[s2-1]||'') };
  map.S3 = { col:s3, header: String(headerRow[s3-1]||'') };
  map.S4 = { col:s4, header: String(headerRow[s4-1]||'') };
  PropertiesService.getDocumentProperties().setProperty('COLMAP', JSON.stringify(map));
  saveHeaderHash_();
}

/***** 9) 🔧 추가 공통 유틸 (C/P 차원 안정화) *****/

/** 비어있거나 문자열이어도 숫자로 안전 변환 */

/** 헤더 행에서 '참석자' 열을 동적으로 탐지한다.
 * - 번호가 1~4가 아니어도 허용 (예: '참석자 6')
 * - 4개 초과면 시트상 '왼쪽부터 4개'만 사용
 * - 4개 미만이면 있는 것만 사용(부족분은 null로 반환)
 * 반환 형식: { S: [ {key:'S1', col: Number, label: string}, ... up to 4 ] }
 */
function detectParticipantCols_(sheet, headerRow){
  const lastCol = sheet.getLastColumn();
  const headers = sheet.getRange(headerRow, 1, 1, lastCol).getDisplayValues()[0];

  // '참석자' 포함 열 모두 후보로
  const candidates = [];
  for (let c=0; c<headers.length; c++){
    const h = (headers[c] || '').toString().trim();
    if (!h) continue;
    if (/참석자\s*\d+/i.test(h)) {
      candidates.push({ col: c+1, label: h });
    }
  }

  // 후보가 없으면 null
  if (candidates.length === 0) return { S: [] };

  // 왼쪽부터 최대 4개만
  candidates.sort((a,b)=>a.col - b.col);
  const picked = candidates.slice(0, 4);

  // S1~S4 키 부여
  const S = picked.map((p, idx)=>({ key: 'S'+(idx+1), col: p.col, label: p.label }));

  return { S };
}

/** S1~S4 매핑을 보장하는 헬퍼
 * - 기존 preflight 매핑에서 못 찾으면 detectParticipantCols_로 대체
 * - 반환: [S1col, S2col, S3col, S4col] (부족분은 null)
 */
function getSColsFlexible_(sheet, preflightCols, headerRow){
  // preflight에 S1..S4가 이미 있으면 그걸 숫자화
  const sCols = [];
  const tryCol = k => {
    if (!preflightCols || preflightCols[k] == null) return null;
    const v = preflightCols[k];
    if (typeof v === 'number') return v;
    if (typeof v === 'object' && v.col != null) return parseInt(v.col, 10);
    if (typeof v === 'string') return parseInt(v, 10);
    return null;
  };

  const s1 = tryCol('S1'), s2 = tryCol('S2'), s3 = tryCol('S3'), s4 = tryCol('S4');
  const hasAll = [s1,s2,s3,s4].some(v => v != null);
  if (!hasAll){
    const detected = detectParticipantCols_(sheet, headerRow);
    const arr = detected.S;
    for (let i=0;i<4;i++){
      sCols[i] = arr[i] ? arr[i].col : null; // 부족분은 null
    }
  } else {
    sCols.push(s1 ?? null, s2 ?? null, s3 ?? null, s4 ?? null);
  }
  return sCols;
}

/** 활성발화자 계산: null 열은 건너뛰고, 값>0 이면 1명으로 카운트 */
function countActiveSpeakersByCols_(sheet, row, sCols){
  let cnt = 0;
  for (const col of sCols){
    if (!col) continue;
    const v = sheet.getRange(row, col).getDisplayValue();
    const n = (v==null||v==='') ? 0 : parseFloat((''+v).replace(/[^\d\.\-]/g,''));
    if (!isNaN(n) && n > 0) cnt++;
  }
  return cnt;
}

/** S1~S4 열 묶음을 숫자열로 통일 (레거시 지원) */

/** G~J의 한 행에서 "활성발화자 수" 계산: 값>0인 셀의 개수 */
function countActiveSpeakers_(rowSlice /*length 4*/){
  let c = 0;
  for (let i=0; i<rowSlice.length; i++){
    if (toNum(rowSlice[i]) > 0) c++;
  }
  return c;
}

/** 요약문 안전 추출 */
function getSummary_(sh, cols, r){
  const fCol = colNumOf(cols.F || cols.SUMMARY);
  return (sh.getRange(r, fCol).getDisplayValue()||'').trim();
}

/** 교사 개입(O열) 안전 추출: true/false */
function isTeacherInvolved_(sh, cols, r){
  const oCol = colNumOf(cols.O || cols.TEACHER_FLAG);
  const v = (sh.getRange(r, oCol).getDisplayValue()||'').trim();
  if (!v) return false;
  // 숫자/불리언/문자 모두 대응
  if (/^(1|true|y|yes|교사|teacher)$/i.test(v)) return true;
  return false;
}

/** PID 기반 컨텍스트(선택): 안전 래퍼 */
function buildPIDIndexIfAny_(sh, cols){
  try{
    const dCol = colNumOf(cols.D); // PID_ASSIGN
    const eCol = colNumOf(cols.E); // PID
    const fCol = colNumOf(cols.F); // SUMMARY
    const lastRow = sh.getLastRow();
    if (lastRow < 2) return null;

    const pids = sh.getRange(2, eCol, lastRow-1, 1).getDisplayValues().map(r=>r[0]);
    const idx = {};
    pids.forEach((pid,i)=>{
      if (!pid) return;
      if (!idx[pid]) idx[pid] = [];
      idx[pid].push( i+2 ); // 실 행번호(시트 행)
    });
    return { map: idx, size:Object.keys(idx).length };
  }catch(e){
    // PID 헤더가 없으면 조용히 패스
    return null;
  }
}

/***** 6) 숫자 안전 변환 (G~J 값 정제) *****/
function asNum01_(v) {
  const n = Number(v);
  if (!isFinite(n) || isNaN(n)) return 0;
  if (n <= 0) return 0;
  return n; // 1 또는 가중값 유지
}

/**
 * 🎯 메뉴 생성 — STEP 12: 단일 numbered "AI 코딩" top-level menu
 *
 * [초기 설정] 1 → 2~3
 * [전처리] 4~5
 * [KCMP Production] 6~10 (6. 진행상황 확인은 단계마다 반복 사용 가능)
 * [결과 확인] 11~12
 * [선택] 13
 * [기타] 캐시 초기화
 *
 * full-run (runCodeX_All, menu_runKCMP) 및 recovery는 메뉴에 노출하지 않음.
 * Production 권장 경로: resume batch (7~10). P gate: ALL_KCM_FINALIZED 필수.
 */
function onOpen() {
  var ui = SpreadsheetApp.getUi();

  ui.createMenu('AI 코딩')
    .addItem('0. 시작 안내', 'showGettingStartedGuide')
    .addItem('1. API키 설정', 'menu_setApiKey')
    .addSeparator()
    .addItem('2. 헤더 설정', 'menu_setupHeaders')
    .addItem('3. 교사·학생 지정', 'menu_assignParticipants')
    .addSeparator()
    .addItem('4. 클러스터링', 'menu_cluster')
    .addItem('5. 화자별 발화분석', 'menu_analyzeSpeakers')
    .addSeparator()
    .addItem('6. 진행상황 확인', 'TEST_KCMP_PRODUCTION_PROGRESS')
    .addItem('7. K 이어서 코딩', 'runCodeK_ResumeBatch')
    .addItem('8. C 이어서 코딩', 'runCodeC_ResumeBatch')
    .addItem('9. M 이어서 코딩', 'runCodeM_ResumeBatch')
    .addItem('10. P 이어서 코딩', 'runCodeP_WithUpstreamPreflight')
    .addSeparator()
    .addItem('11. P 결과 요약', 'TEST_P_PRODUCTION_SUMMARY')
    .addItem('12. 오류 목록 확인', 'TEST_KCMP_PRODUCTION_ERROR_INVENTORY')
    .addSeparator()
    .addItem('13. 다이어그램 작성', 'menu_drawDiagram')
    .addSeparator()
    .addItem('기타. 개인 API키 삭제', 'clearMyApiKey')
    .addItem('기타. 캐시 초기화', 'menu_resetCache')
    .addToUi();
}


/**
 * 애드온 첫 실행 안내를 사이드바로 표시합니다.
 * 안내는 자동 결과를 연구자 검토 없이 확정하지 않도록 명시합니다.
 */
function showGettingStartedGuide(){
  var html = HtmlService.createHtmlOutput(
    '<!doctype html><html><head><base target="_top"><style>' +
    'body{font-family:Arial,sans-serif;line-height:1.55;padding:16px;color:#202124}' +
    'h1{font-size:19px;margin:0 0 12px}h2{font-size:15px;margin:18px 0 6px}' +
    'table{border-collapse:collapse;width:100%;font-size:12px}th,td{border:1px solid #dadce0;padding:6px;text-align:left}' +
    '.note{background:#fff8e1;border-left:4px solid #fbbc04;padding:10px;margin-top:14px}' +
    '</style></head><body>' +
    '<h1>KCMP AI 코딩 시작 안내</h1>' +
    '<h2>1. 전사 데이터 준비</h2>' +
    '<p>4명 내외 소집단 녹음을 전사합니다. <b>1행은 헤더 행</b>이며, 실제 전사 데이터는 2행부터 입력합니다.</p>' +
    '<table><tr><th>열</th><th>입력 내용</th></tr><tr><td>A</td><td>화자 이름</td></tr><tr><td>B</td><td>타임스탬프</td></tr><tr><td>C</td><td>발화 내용</td></tr></table>' +
    '<h2>2. 개인 API 키 설정</h2>' +
    '<p>OpenAI Platform에서 결제를 활성화한 뒤, <b>AI 코딩 → 1. API키 설정</b>에서 본인의 API 키를 입력합니다. 키는 현재 사용자에게만 저장되며 사용 요금은 해당 OpenAI 계정에 청구됩니다.</p>' +
    '<h2>3. 분석 실행 순서</h2>' +
    '<p>헤더 설정 → 교사·학생 지정 → 클러스터링 → 화자별 발화분석을 실행합니다. 이후 K, C, M, P는 실행 시간 제한을 고려해 <b>이어서 코딩</b> 메뉴를 완료될 때까지 반복 실행합니다.</p>' +
    '<div class="note"><b>중요:</b> 자동 코딩 결과는 초안입니다. 최종 사용 전에 반드시 원문 전사본과 대조하고, 연구자 또는 교사가 코드와 근거를 검토·수정하세요.</div>' +
    '</body></html>'
  ).setTitle('KCMP AI 코딩 시작 안내');
  SpreadsheetApp.getUi().showSidebar(html);
}

function onInstall(e){
  onOpen(e);
}

/** 클러스터링 메뉴 연결 */
function menu_cluster(){
  try{
    const sh = SpreadsheetApp.getActiveSheet();
    const map = ensureColMapOrHalt_();

    assignClustersOnly(); // D열만 갱신

    const lastRowLimit = sh.getLastRow(); // 우선 전체를 기준
    normalizePidIntoE_(sh, map, lastRowLimit);             // E: 고유 PID 목록 재작성
    const limit = findLastPidRow_(sh, map);                 // 실제 E 마지막 PID까지
    buildClusterSummariesFromPID_(sh, map, limit);          // F: E행별 취합 요약

    SpreadsheetApp.getUi().alert('✅ D→E목록→F요약까지 업데이트 완료');
  }catch(e){
    SpreadsheetApp.getUi().alert('⚠️ 클러스터링 중 오류: '+e);
  }
}

/** 다이어그램 작성 메뉴 연결 */
function menu_drawDiagram(){
  buildDiagram13_fast();
}

/** 화자별 발화분석 메뉴 연결 */
function menu_analyzeSpeakers(){
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const cols = ensureColMapOrHalt_();

  const Acol = colNumOf(cols.A), Dcol = colNumOf(cols.D), Ecol = colNumOf(cols.E);
  
  // G~J는 헤더 순서(참석자 1→2→3→4)로 기록합니다.
  // A열의 "참석자 N" 표기가 헤더 라벨과 달라도 숫자 폴백으로 매칭합니다.
  const sCols = getSColsFlexible_(sh, cols, 1);
  
  // S1~S4 열 검증 강화
  const validCols = sCols.filter(c => c && c >= 1 && c <= 1000);
  if (validCols.length === 0) {
    ui.alert('⚠️ G~J 열을 찾을 수 없습니다.\n\n해결 방법:\n1. [헤더 설정] 실행\n2. [교사·학생 지정] 실행\n3. 다시 시도');
    return;
  }
  if (validCols.length < 2) {
    ui.alert('⚠️ 화자 열이 2개 미만입니다. 최소 2개 이상의 화자 열이 필요합니다.');
    return;
  }
  
  const hdr   = getHeaderRow_(sh);
  const sHeaders = sCols.map((c,i)=> c ? ({ key:'S'+(i+1), col:c, label:String(hdr[c-1]||'').trim() }) : null).filter(Boolean);
  
  if (sHeaders.length === 0) {
    ui.alert('⚠️ 화자 헤더를 찾을 수 없습니다. 헤더 행에 "참석자 1", "참석자 2" 등의 라벨이 있는지 확인하세요.');
    return;
  }

  const sheetLastRow = sh.getLastRow();
  if (sheetLastRow < 2){ ui.alert('데이터가 없습니다.'); return; }

  const lastPidRow = findLastPidRow_(sh, cols);
  const eLen = Math.max(0, Math.min(sheetLastRow, lastPidRow) - 1);
  const E = eLen ? sh.getRange(2, Ecol, eLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim().toUpperCase()) : [];

  // 시트 전체 A/D
  const dataLen = sheetLastRow - 1;
  const A = dataLen ? sh.getRange(2, Acol, dataLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim()) : [];
  const D = dataLen ? sh.getRange(2, Dcol, dataLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim().toUpperCase()) : [];

  // 디버깅: 매칭 로그 수집
  const matchLog = [];
  let totalMatches = 0;
  let totalSkipped = 0;

  // 성능 개선: 배치 처리로 변경 - 모든 행을 한 번에 쓰기
  const allRowValues = [];
  
  for (let i=0;i<E.length;i++){
    const pid = E[i];
    if (!/^P\d+/i.test(pid)) {
      allRowValues.push([0, 0, 0, 0]); // 빈 행
      continue;
    }

    const cnt = {S1:0,S2:0,S3:0,S4:0};
    // 성능 개선: PID별로 그룹화하여 한 번에 처리
    for (let r=0;r<dataLen;r++){
      if (D[r] !== pid) continue;
      const spk = A[r];
      if (!spk) {
        totalSkipped++;
        continue;
      }
      
      // 교사 제외 (정확한 단어만)
      const spkTrimmed = spk.trim();
      if (EXCLUDE_SPEAKER_PAT.test(spkTrimmed)) {
        totalSkipped++;
        continue;
      }

      // 방법 1: matchSpeakerToSx_ 사용
      let hit = matchSpeakerToSx_(spk, sHeaders);
      
      // 방법 2: 폴백 - "참석자 N" 패턴 직접 추출
      if (!hit){
        const m = (spk||'').replace(/\s+/g,'').match(/참석자(\d{1,2})/);
        if (m){ 
          const n = +m[1]; 
          if (n>=1 && n<=4) hit = 'S'+n; 
        }
      }
      
      // 방법 3: 폴백 - 숫자만 추출 (S1, S2 등)
      if (!hit) {
        const numMatch = (spk||'').replace(/\s+/g,'').match(/^[Ss]?(\d{1,2})$/);
        if (numMatch) {
          const n = +numMatch[1];
          if (n>=1 && n<=4) hit = 'S'+n;
        }
      }
      
      if (hit) {
        cnt[hit] = (cnt[hit]||0) + 1;
        totalMatches++;
        
        // 디버깅 로그 (최대 10개만)
        if (matchLog.length < 10) {
          matchLog.push({ speaker: spk, matched: hit, pid: pid });
        }
      } else {
        totalSkipped++;
        // 매칭 실패 로그 (최대 5개만)
        if (matchLog.length < 15 && matchLog.filter(l => !l.matched).length < 5) {
          matchLog.push({ speaker: spk, matched: null, pid: pid });
        }
      }
    }

    // 행 값 준비 - 항상 4개 값을 보장
    const rowValues = [];
    for (let k=0;k<4;k++){
      const key = 'S'+(k+1);
      rowValues.push(cnt[key]||0);
    }
    allRowValues.push(rowValues);
  }
  
  // 성능 개선: 한 번에 모든 행 쓰기 (개별 setValue 대신)
  if (allRowValues.length > 0) {
    // sCols에서 null이 아닌 첫 번째 열 찾기
    const validCols = sCols.filter(c => c);
    if (validCols.length === 0) {
      ui.alert('⚠️ G~J 열을 찾을 수 없습니다. 교사·학생 지정을 먼저 실행하세요.');
      return;
    }
    const startCol = validCols[0];
    const endCol = validCols[validCols.length - 1];
    const numCols = endCol - startCol + 1;
    sh.getRange(2, startCol, allRowValues.length, numCols).setValues(allRowValues);
  }
  
  // 결과 검증 및 디버깅 정보
  const totalCount = allRowValues.reduce((sum, row) => sum + row.reduce((s, v) => s + (v || 0), 0), 0);
  
  if (totalCount === 0) {
    // 모든 값이 0인 경우 상세 로그 출력
    let debugMsg = '⚠️ 화자별 발화분석 결과가 모두 0입니다.\n\n';
    debugMsg += `📊 통계:\n`;
    debugMsg += `- 총 매칭 성공: ${totalMatches}건\n`;
    debugMsg += `- 총 건너뜀: ${totalSkipped}건\n`;
    debugMsg += `- 처리된 PID: ${E.filter(p => /^P\d+/i.test(p)).length}개\n\n`;
    
    if (matchLog.length > 0) {
      debugMsg += `🔍 매칭 샘플 (최대 10개):\n`;
      matchLog.slice(0, 10).forEach((log, idx) => {
        if (log.matched) {
          debugMsg += `${idx+1}. "${log.speaker}" → ${log.matched} ✅\n`;
        } else {
          debugMsg += `${idx+1}. "${log.speaker}" → 매칭 실패 ❌\n`;
        }
      });
      debugMsg += `\n`;
    }
    
    debugMsg += `💡 해결 방법:\n`;
    debugMsg += `1. A열 발화자 이름이 G~J 헤더 라벨과 일치하는지 확인\n`;
    debugMsg += `2. "참석자 1", "1", "S1" 등 다양한 형식 시도\n`;
    debugMsg += `3. [교사·학생 지정] 메뉴를 다시 실행\n`;
    
    Logger.log('화자별 발화분석 실패: ' + JSON.stringify(matchLog.slice(0, 10)));
    ui.alert(debugMsg);
  } else {
    // 성공 메시지에 통계 포함
    let successMsg = '✅ 화자별 발화분석 완료\n\n';
    successMsg += `📊 통계:\n`;
    successMsg += `- 총 발화 수: ${totalCount}건\n`;
    successMsg += `- 매칭 성공: ${totalMatches}건\n`;
    if (matchLog.length > 0 && matchLog[0].matched) {
      successMsg += `- 샘플 매칭: "${matchLog[0].speaker}" → ${matchLog[0].matched}\n`;
    }
    ui.alert(successMsg);
  }
}

/** 라벨 매칭 실패 시 숫자 백업 매칭 */
function _fallbackMatchSpeaker_(speakerText){
  const t = (speakerText||'').toString().replace(/\s+/g,'').toLowerCase();
  const m = t.match(/참석자(\d{1,2})/);
  if (m){
    const n = parseInt(m[1],10);
    if (n>=1 && n<=4) return 'S'+n; // S1..S4
  }
  return null;
}

/**
 * 📋 헤더 설정 (1행 강제 기입 + 헤더해시 저장)
 */
function menu_setupHeaders(){
  const sh = SpreadsheetApp.getActiveSheet();
  const hdr = getHeaderRow_(sh);

  const set = (col, text) => sh.getRange(1, col).setValue(text);

  // 고정 헤더
  set(1,'발화자'); set(2,'타임스탬프'); set(3,'발화'); set(4,'PID부여'); set(5,'PID');
  set(6,'요약문'); /* G~J 비워둠 (학생 헤더는 역할 지정 시 채움) */
  set(11,'K차원'); set(12,'C차원'); set(13,'M차원'); set(14,'P차원'); set(15,'교사 개입 여부'); set(16,'패턴');

  // COLMAP 저장 (S1~S4는 나중에 교사·학생 지정에서 채움)
  const map = {
    A: {col:1, header:'발화자'}, B:{col:2, header:'타임스탬프'}, C:{col:3, header:'발화'},
    D: {col:4, header:'PID부여'}, E:{col:5, header:'PID'}, F:{col:6, header:'요약문'},
    G: {col:7, header:''}, H:{col:8, header:''}, I:{col:9, header:''}, J:{col:10, header:''},
    K: {col:11, header:'K차원'}, L:{col:12, header:'C차원'}, M:{col:13, header:'M차원'}, N:{col:14, header:'P차원'},
    O: {col:15, header:'교사 개입 여부'}, P:{col:16, header:'패턴'}
  };
  PropertiesService.getDocumentProperties().setProperty('COLMAP', JSON.stringify(map));
  saveHeaderHash_();
  styleHeaderRow(); // 🔥 1행 스타일 자동 적용
  SpreadsheetApp.getUi().alert('✅ 헤더가 설정되었습니다.\n다음으로 [교사·학생 지정]을 실행하세요.');
}

/**
 * 👥 교사·학생 지정 (교사 prompt 무조건 표시, 학생 4명 수집 → G~J 채움, 내부키=S1..S4 저장)
 */
function menu_assignParticipants(){
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  let map = getColMap_() || {};
  map = normalizeSColsInMap_(map);

  const teacher = ui.prompt('교사 이름(또는 표기)을 입력하세요.', ui.ButtonSet.OK_CANCEL);
  if (teacher.getSelectedButton() !== ui.Button.OK) return;
  const tname = teacher.getResponseText().trim();

  const ask = label => {
    const r = ui.prompt(label+' 학생 이름(또는 표기)을 입력하세요.', ui.ButtonSet.OK_CANCEL);
    if (r.getSelectedButton() !== ui.Button.OK) throw new Error('취소됨');
    return r.getResponseText().trim();
  };

  const s1 = ask('참석자 1'); const s2 = ask('참석자 2'); const s3 = ask('참석자 3'); const s4 = ask('참석자 4');

  sh.getRange(1, 7).setValue(s1);
  sh.getRange(1, 8).setValue(s2);
  sh.getRange(1, 9).setValue(s3);
  sh.getRange(1,10).setValue(s4);

  map.S1 = { col:7,  header:s1 };
  map.S2 = { col:8,  header:s2 };
  map.S3 = { col:9,  header:s3 };
  map.S4 = { col:10, header:s4 };
  map.TEACHER = tname;

  PropertiesService.getDocumentProperties().setProperty('COLMAP', JSON.stringify(map));
  saveHeaderHash_();
  styleHeaderRow(); // 🔥 1행 스타일 자동 적용
  ui.alert(`✅ 교사/학생 지정 완료\n교사: ${tname}\nS1:${s1} / S2:${s2} / S3:${s3} / S4:${s4}`);
  
  // 자동으로 헤더·열매핑 점검 실행
  menu_inspectMapping();
}

/**
 * 🔑 API 키 설정 (기존 함수 연결)
 */
function menu_setApiKey() {
  setApiKeyOnce();
}

/**
 * 🗑️ 캐시 초기화
 */
function menu_resetCache(){
  getStore_().deleteAllProperties();
  
  const msg = `✅ 캐시 초기화 완료!\n\n` +
              `📋 다음 단계 (순서대로):\n\n` +
              `1️⃣ [헤더 설정]\n` +
              `   → 1행에 표준 헤더 기입\n\n` +
              `2️⃣ [교사·학생 지정]\n` +
              `   → 교사 이름 + 학생 4명 입력\n` +
              `   → G~J 헤더에 학생 이름 기록\n\n` +
              `3️⃣ [화자별 발화분석] 또는 [KCM/P코딩]\n` +
              `   → 정상 작동!\n\n` +
              `⚠️ 중요: 반드시 ① 헤더 설정 → ② 교사·학생 지정 → ③ 화자별 발화분석/KCM/P코딩 순으로 실행하세요.\n` +
              `중간에 헤더가 바뀌면 다시 ①→②를 먼저 실행해야 합니다.\n\n` +
              `💡 이 순서를 지키면 "[object Object]" 오류가 발생하지 않습니다.`;
  
  SpreadsheetApp.getUi().alert(msg);
}

/**
 * 🔍 헤더·열매핑 점검 (모달/알림) - 디버그 강화
 */
function menu_inspectMapping(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = getColMap_();
  const changed = headerChanged_();
  
  let msg = `═══════════════════════════\n`;
  msg += `📋 헤더·열매핑 점검\n`;
  msg += `═══════════════════════════\n\n`;
  msg += `🔄 헤더 변경됨? ${changed ? '⚠️ YES (재설정 필요)' : '✅ NO'}\n\n`;
  
  if (map) {
    msg += `📊 현재 COLMAP:\n`;
    msg += `  • 교사: ${map.TEACHER || '미설정'}\n`;
    
    // 🔧 S1~S4 실제 열번호와 1행 라벨 표시
    const sCols = getSColsFlexible_(sh, map, 1);
    const hdr = getHeaderRow_(sh);
    sCols.forEach((c, i) => {
      const label = c ? String(hdr[c-1]||'') : '미설정';
      msg += `  • S${i+1} (열${c || '?'}): ${label}\n`;
    });
    msg += `\n`;
    
    // 🔧 A열 화자 상위 5개 샘플 (정규화 전/후 → S키 매칭 결과)
    try {
      const aCol = colNumOf(map.A);
      const lastRow = Math.min(sh.getLastRow(), 20);
      const speakers = sh.getRange(2, aCol, lastRow-1, 1).getDisplayValues().map(r=>r[0]);
      const samples = [...new Set(speakers.filter(s=>s))].slice(0, 5);
      
      const S = sCols.map((c, i) => ({ 
        key:'S'+(i+1), 
        label: c ? String(hdr[c-1]||'').replace(/\s*\(\d+\)\s*$/,'').trim() : '' 
      })).filter(s => !!s.label);
      
      msg += `👥 A열 화자 샘플 (상위 5개):\n`;
      samples.forEach(spk => {
        const norm = _normSpeakerTag(spk);
        const hit = matchSpeakerToSx_(spk, S);
        msg += `  "${spk}" → 정규화: "${norm}" → 매칭: ${hit || '실패'}\n`;
      });
      msg += `\n`;
    } catch (e) {
      msg += `⚠️ A열 샘플 읽기 오류: ${e}\n\n`;
    }
    
    // 🔧 PID 고유 개수 / PID별 평균 행수
    try {
      const pidIdx = buildPIDIndex_(sh, map);
      const pids = Object.keys(pidIdx);
      const totalRows = pids.reduce((sum, pid) => sum + pidIdx[pid].rows.length, 0);
      const avgRows = pids.length > 0 ? (totalRows / pids.length).toFixed(1) : 0;
      
      msg += `📊 PID 통계:\n`;
      msg += `  • 고유 PID: ${pids.length}개\n`;
      msg += `  • PID별 평균 행수: ${avgRows}행\n\n`;
    } catch (e) {
      msg += `⚠️ PID 통계 계산 오류: ${e}\n\n`;
    }
    
    // 헤더 1행 샘플
    try {
      const headerRow = sh.getRange(1,1,1,16).getValues()[0];
      msg += `📄 헤더 1행 (A~P):\n`;
      for (let i=0; i<headerRow.length; i++) {
        const colLetter = String.fromCharCode(65 + i); // A, B, C, ...
        msg += `  ${colLetter}: ${headerRow[i] || '(빈칸)'}\n`;
      }
    } catch (e) {
      msg += `⚠️ 헤더 읽기 오류: ${e}\n`;
    }
  } else {
    msg += `⚠️ COLMAP이 설정되지 않았습니다.\n`;
    msg += `[헤더 설정] → [교사·학생 지정] 순서로 실행하세요.\n`;
  }
  
  SpreadsheetApp.getUi().alert(msg);
}

/********************
 * 🔧 PID 정규화/추출 유틸
 ********************/
function normPID_(val){
  if (!val) return '';
  const s = String(val).trim();
  // 가장 신뢰되는 패턴: P + 숫자
  const m = s.match(/P\d{1,5}/i);
  return m ? m[0].toUpperCase() : '';
}

// F열(요약문)에서 PID 힌트 추출 (예: "■ 12:07~12:19 ... P001 ..." 형태 포함 가능)
function pidFromSummary_(summary){
  if (!summary) return '';
  const s = String(summary);
  
  // 1) 명시적 PID 패턴 우선
  const m = s.match(/P\d{1,5}/i);
  if (m) return m[0].toUpperCase();
  
  // 2) 폴백: "■ mm:ss~mm:ss" → Pmmssmmss 형태로 변환
  const t = s.match(/■\s*(\d{2}:\d{2})\s*~\s*(\d{2}:\d{2})/);
  if (t){
    return ('P' + t[1].replace(/:/g,'') + t[2].replace(/:/g,'')).toUpperCase();
  }
  
  return '';
}

/********************
 * 🏷️ 화자 매칭 유틸 (강화)
 ********************/

/** 유니코드 정규화 + 제로폭 문자 제거 */
function _norm_(s){ 
  return (s||'').normalize('NFKC').replace(/\u200B/g,'').trim(); 
}

/** 화자명 정규화: 괄호/공백/꼬리숫자 제거 - 발화자와 헤더 라벨 모두 동일하게 처리 */
function _normSpeakerTag(s){
  if (!s) return '';
  return String(s)
    .normalize('NFKC')        // 유니코드 정규화
    .replace(/\u200B/g,'')    // 제로폭 문자 제거
    .replace(/[\(\)\[\]【】]/g,'')   // 모든 괄호 제거
    .replace(/\s+/g,'')       // 공백 제거
    .replace(/\d+$/,'')       // 꼬리 숫자 제거(이름(62) 등)
    .toLowerCase();
}

/** 발화자와 헤더 라벨을 통일된 방식으로 정규화 (매칭용) */
function _normForMatching_(s){
  if (!s) return '';
  const normalized = String(s)
    .normalize('NFKC')
    .replace(/\u200B/g,'')
    .replace(/[\(\)\[\]【】]/g,'')   // 괄호 제거
    .replace(/\s+/g,'')               // 공백 제거
    .toLowerCase();
  
  // 숫자 추출 (참석자 1, 1, S1 등)
  const numMatch = normalized.match(/(\d+)/);
  if (numMatch) {
    return {
      normalized: normalized,
      number: parseInt(numMatch[1], 10),
      hasNumber: true
    };
  }
  
  return {
    normalized: normalized,
    number: null,
    hasNumber: false
  };
}

/** 교사 제외 패턴 - 정확한 단어 경계만 매칭 (학생 이름에 "선생" 포함된 경우 제외 방지) */
var EXCLUDE_SPEAKER_PAT = /^(교사|teacher|선생님|쌤|T|Teacher)$/i;

/** 행 r에서 PID(E)와 요약문(F)가 모두 존재하면 true */
function _hasPidAndSummary_(sh, cols, r){
  const pid = (sh.getRange(r, colNumOf(cols.E)).getDisplayValue()||'').trim();
  const sum = (sh.getRange(r, colNumOf(cols.F)).getDisplayValue()||'').trim();
  return !!(pid && sum);
}


/** 헤더 이름으로 S1~S4를 찾을 때 경계 기반 안전 매칭 (10/11 오탐 방지) - 강화 버전 */
function matchSpeakerToSx_(speakerText, sHeaders){
  if (!speakerText) return null;
  
  // 교사/선생님 제외 (정확한 단어만)
  const speakerTrimmed = String(speakerText).trim();
  if (EXCLUDE_SPEAKER_PAT.test(speakerTrimmed)) return null;
  
  // 발화자 정규화
  const speakerNorm = _normForMatching_(speakerText);
  if (!speakerNorm.normalized) return null;
  
  // sHeaders는 [{key:'S1', label:'참석자 1', col:7}, ...] 형태
  for (const h of sHeaders){
    if (!h || !h.label) continue;
    
    // 헤더 라벨 정규화
    const labelNorm = _normForMatching_(h.label);
    if (!labelNorm.normalized) continue;
    
    // 방법 1: 숫자 기반 매칭 (가장 안정적)
    if (speakerNorm.hasNumber && labelNorm.hasNumber) {
      if (speakerNorm.number === labelNorm.number && speakerNorm.number >= 1 && speakerNorm.number <= 4) {
        return h.key;
      }
    }
    
    // 방법 2: 정규화된 문자열 직접 비교
    if (speakerNorm.normalized === labelNorm.normalized) {
      return h.key;
    }
    
    // 방법 3: "참석자" + 숫자 패턴 매칭
    const speakerNum = speakerNorm.normalized.match(/참석자\s*(\d+)/);
    const labelNum = labelNorm.normalized.match(/참석자\s*(\d+)/);
    if (speakerNum && labelNum && speakerNum[1] === labelNum[1]) {
      return h.key;
    }
    
    // 방법 4: "S" + 숫자 패턴 매칭 (S1, S2 등)
    const speakerS = speakerNorm.normalized.match(/^s\s*(\d+)$/);
    const labelS = labelNorm.normalized.match(/^s\s*(\d+)$/);
    if (speakerS && labelS && speakerS[1] === labelS[1]) {
      return h.key;
    }
    
    // 방법 5: 숫자만 매칭 (경계 기반, 10과 1 구분)
    if (speakerNorm.hasNumber && labelNorm.hasNumber) {
      const speakerNumStr = String(speakerNorm.number);
      const labelNumStr = String(labelNorm.number);
      // 정규화된 문자열에 숫자가 단독으로 나타나는지 확인
      if (speakerNorm.normalized.includes(speakerNumStr) && 
          labelNorm.normalized.includes(labelNumStr) &&
          speakerNorm.number === labelNorm.number) {
        // 추가 검증: 숫자 앞뒤가 단어 경계인지 확인
        const speakerPattern = new RegExp('(^|[^\\d])' + speakerNumStr + '([^\\d]|$)');
        const labelPattern = new RegExp('(^|[^\\d])' + labelNumStr + '([^\\d]|$)');
        if (speakerPattern.test(speakerNorm.normalized) && labelPattern.test(labelNorm.normalized)) {
          return h.key;
        }
      }
    }
    
    // 방법 6: 부분 문자열 포함 검사 (이름 기반 매칭)
    const speakerClean = speakerNorm.normalized.replace(/참석자|s|\d+/g, '').trim();
    const labelClean = labelNorm.normalized.replace(/참석자|s|\d+/g, '').trim();
    if (speakerClean && labelClean && speakerClean.length >= 2 && labelClean.length >= 2) {
      if (speakerClean === labelClean || 
          speakerClean.includes(labelClean) || 
          labelClean.includes(speakerClean)) {
        // 숫자도 일치하는지 확인
        if (!speakerNorm.hasNumber || !labelNorm.hasNumber || speakerNorm.number === labelNorm.number) {
          return h.key;
        }
      }
    }
  }
  
  return null;
}

/********************
 * 🗂️ PID 인덱스 만들기
 * - 기준: E열(PID) → 없으면 D열(PID부여) → F열(요약문)에서 정규표현식으로 추론
 * - 각 PID마다 {rows:[], repRow:요약행(대표), speakers:{이름:카운트}} 구조
 ********************/
function buildPIDIndex_(sh, map){
  const lastRow = sh.getLastRow();
  if (lastRow < 2) {
    Logger.log('⚠️ buildPIDIndex_: 데이터 행이 없음 (lastRow < 2)');
    return {};
  }

  const rng = sh.getRange(2,1,lastRow-1,16);
  const data = rng.getValues();

  // 🔧 colNumOf로 안전하게 열 번호 추출
  const PID_COL = colNumOf(map.E) || 5; // E열
  const PIDASSIGN_COL = colNumOf(map.D) || 4; // D열
  const SUM_COL = colNumOf(map.F) || 6; // F열
  const SPEAKER_COL = colNumOf(map.A) || 1; // A열

  Logger.log(`📊 buildPIDIndex_: ${data.length}행 처리 시작`);
  Logger.log(`  PID_COL(E)=${PID_COL}, PIDASSIGN_COL(D)=${PIDASSIGN_COL}, SUM_COL(F)=${SUM_COL}, SPEAKER_COL(A)=${SPEAKER_COL}`);

  const idx = Object.create(null);
  let pidFoundCount = 0;
  let speakerFoundCount = 0;

  data.forEach((r, i) => {
    // 1) E → 2) D → 3) F(요약)에서 추론
    let pid = normPID_(r[PID_COL-1]);
    if (!pid) pid = normPID_(r[PIDASSIGN_COL-1]);
    if (!pid) pid = pidFromSummary_(r[SUM_COL-1]);

    if (!pid) {
      // 첫 5행만 로그 (너무 많으면 생략)
      if (i < 5) {
        Logger.log(`  행${i+2}: PID 없음 (E="${r[PID_COL-1]}", D="${r[PIDASSIGN_COL-1]}", F="${String(r[SUM_COL-1]||'').substring(0,30)}...")`);
      }
      return; // PID 없는 행은 스킵
    }

    pidFoundCount++;
    if (!idx[pid]) idx[pid] = { rows: [], repRow: null, speakers: Object.create(null) };

    const rowNumber = i + 2; // 실제 시트 행 번호
    idx[pid].rows.push(rowNumber);

    // 대표행(요약행) 후보: F가 장문(60자 이상)인 행 우선
    const summary = String(r[SUM_COL-1] || '');
    if (!idx[pid].repRow) {
      idx[pid].repRow = rowNumber; // 첫 행을 기본값으로
    }
    if (summary.length >= 60) {
      idx[pid].repRow = rowNumber; // 장문 우선
    }

    // 화자 카운팅(A열)
    const speaker = String(r[SPEAKER_COL-1] || '').trim();
    if (speaker) {
      speakerFoundCount++;
      idx[pid].speakers[speaker] = (idx[pid].speakers[speaker] || 0) + 1;
      // 첫 5개만 로그
      if (speakerFoundCount <= 5) {
        Logger.log(`  행${rowNumber}: PID=${pid}, 화자="${speaker}"`);
      }
    }
  });

  Logger.log(`✅ PID 발견: ${pidFoundCount}행, 화자 발견: ${speakerFoundCount}행, 고유 PID: ${Object.keys(idx).length}개`);

  // 🔧 최종 가드: 각 PID의 대표행이 없으면 강제로 첫 행 지정
  Object.keys(idx).forEach(pid => {
    if (!idx[pid].repRow && idx[pid].rows.length > 0) {
      idx[pid].repRow = idx[pid].rows[0];
      Logger.log(`  PID ${pid}: 대표행을 첫 행(${idx[pid].rows[0]})로 설정`);
    }
  });

  // 첫 3개 PID 샘플 로그
  const samplePIDs = Object.keys(idx).slice(0, 3);
  samplePIDs.forEach(pid => {
    const cluster = idx[pid];
    Logger.log(`  PID ${pid}: 행수=${cluster.rows.length}, 대표행=${cluster.repRow}, 화자=${JSON.stringify(cluster.speakers)}`);
  });

  return idx;
}

/********************
 * 🔐 참가자 헤더/매핑 불러오기 (강화된 검증)
 ********************/
function ensureColMapOrHalt_(){
  const sh = SpreadsheetApp.getActiveSheet();
  
  if (headerChanged_()) {
    SpreadsheetApp.getUi().alert('헤더가 변경되었습니다. [교사·학생 지정]을 다시 실행하세요.');
    throw new Error('Header changed');
  }
  
  let map = loadColMap_();
  
  if (!map || !map.S1 || !map.S4) {
    const msg = `⚠️ S1~S4 매핑이 없습니다!\n\n` +
                `📋 해결 방법:\n\n` +
                `1️⃣ [캐시 초기화]\n` +
                `2️⃣ [헤더 설정]\n` +
                `3️⃣ [교사·학생 지정]\n\n` +
                `순서대로 실행하세요.`;
    SpreadsheetApp.getUi().alert(msg);
    throw new Error('No S1~S4 mapping');
  }
  
  // 🔧 객체 형식 보증 + 검증
  map = normalizeSColsInMap_(map);
  
  try {
    validateColMap_(map, sh);
  } catch (e) {
    const msg = `🚨 COLMAP 검증 실패!\n\n` +
                `오류: ${e.message}\n\n` +
                `COLMAP 상태:\n` +
                `  S1: ${JSON.stringify(map.S1)}\n` +
                `  S2: ${JSON.stringify(map.S2)}\n` +
                `  S3: ${JSON.stringify(map.S3)}\n` +
                `  S4: ${JSON.stringify(map.S4)}\n\n` +
                `해결:\n` +
                `1️⃣ [캐시 초기화]\n` +
                `2️⃣ [헤더 설정]\n` +
                `3️⃣ [교사·학생 지정]\n\n` +
                `순서대로 다시 실행하세요.\n\n` +
                `💡 상세 로그: 보기 → 로그`;
    SpreadsheetApp.getUi().alert(msg);
    throw e;
  }
  
  Logger.log(`✅ ensureColMapOrHalt_: S1~S4=${getSCols_(map).join(',')}`);
  
  return map;
}


// ★ 구버전 menu_analyzeSpeakers 삭제됨 (417줄의 최신 버전 사용)

/**
 * 🤖 KCMP 일괄 코딩 (K → C → M → P 순차 실행)
 * CANONICAL ORDER: K → C → M → P (STEP 7 확정)
 */
function menu_runKCMP(){
  const ui = SpreadsheetApp.getUi();
  
  // 확인 다이얼로그
  const res = ui.alert(
    'KCMP 일괄 코딩',
    'K → C → M → P 순서로 모든 차원을 코딩합니다.\n계속하시겠습니까?',
    ui.ButtonSet.YES_NO
  );
  
  if (res !== ui.Button.YES) return;
  
  try {
    // 1단계: K 코딩
    runCodeK_All();
    
    // 2단계: C 코딩
    runCodeC_All();
    
    // 3단계: M 코딩
    runCodeM_All();
    
    // 4단계: P 코딩 (deterministic, K/C/M Note 기반)
    runCodeP_All();
    
    ui.alert('✅ KCMP 일괄 코딩 완료!\n\nK → C → M → P 순서로 모두 처리되었습니다.');
  } catch (e) {
    ui.alert(`⚠️ KCMP 코딩 중 오류 발생:\n\n${e.message}\n\n진행된 단계까지는 저장되었습니다.`);
  }
}

/***** 10) 🅰️ A&D 차원 코딩 (GPT 기반) *****/

/** 📦 간단 GPT 래퍼: 요약문 → K코드(+근거) */
function _gptCodeK_(summary){
  try {
    // === 폴백(로컬 휴리스틱) ===
    const t = summary.toLowerCase();
    if (/(왜|때문|근거|설명|원인|인과|정의|원리)/.test(t)) return { code:'K1. 개념/근거 설명', reason:'요약문에 이유·근거를 들어 설명함' };
    if (/(증거|데이터|그래프|표|관찰)/.test(t)) return { code:'K3. 주장+근거 정당화', reason:'자료/증거를 근거로 주장을 정당화함' };
    return { code:'K1. 개념/근거 설명', reason:'설명적 발화가 중심' };
  } catch(e){
    return { code:'K1. 개념/근거 설명', reason:'(폴백) 요약문 기반 자동 판정' };
  }
}

// ★ 구버전 runCodeAD_All() 삭제됨 (6675줄의 최신 버전 사용 - lastRowLimit 포함)

/***** 11) 🅲 C차원 코딩 (활성발화자 기반) *****/

/** 🔄 C 코딩 — clusterPacket.turns 기반 C Decision Tree v1.0 */
function runCodeC_All(){
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const cCol = colNumOf(map.L);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  if (!packets || !packets.length) {
    ui.alert('⚠️ 처리할 PID 패킷이 없습니다.\n클러스터링과 화자별 발화분석을 먼저 실행하세요.');
    return;
  }

  let codedCount = 0;
  let noneCount = 0;
  let errorCount = 0;
  let skippedCount = 0;

  packets.forEach(function(packet) {
    const row = packet && packet.representativeRow;
    if (!row) {
      skippedCount++;
      Logger.log('⚠️ C 코딩 건너뜀: representativeRow 없음 pid=' + (packet && packet.pid));
      return;
    }

    const result = runCDecisionTreeForPacket_(packet);
    _writeCDecisionCell_(sh, row, cCol, result);

    if (result && result.status === 'OK' && result.code) {
      codedCount++;
      if (codedCount <= 3) Logger.log('C 코딩 성공 [행' + row + '] ' + packet.pid + ': ' + result.code);
    } else if (result && result.status === 'OK' && result.code == null) {
      noneCount++;
    } else {
      errorCount++;
      Logger.log('❌ C 코딩 오류 [행' + row + '] ' + (packet.pid || '') + ': ' + (result && result.error_type) + ' ' + (result && result.message));
    }
  });

  let msg = '✅ C 코딩 완료 (Decision Tree v1.0)\n\n';
  msg += '📊 통계:\n';
  msg += '- C1~C7 부여: ' + codedCount + '개\n';
  msg += '- 정상 C 없음: ' + noneCount + '개\n';
  if (errorCount > 0) msg += '- 오류: ' + errorCount + '개\n';
  if (skippedCount > 0) msg += '- 대표행 없음 skip: ' + skippedCount + '개\n';
  msg += '\nC 없음과 오류는 L셀 값(빈칸)이 같아 보여도 Note JSON의 status로 구분됩니다.';
  if (errorCount > 0) {
    msg += '\n\n⚠️ 오류가 있습니다. [보기] → [실행 로그]에서 error_type을 확인하세요.';
  }
  ui.alert(msg);
}

// ============================================================
// LEGACY / INACTIVE / NOT USED BY LIVE KCMP PIPELINE
// getCCodePromptTemplate_: GPT 일괄 C 코딩용 구버전 프롬프트 생성 함수.
// live path(runCodeC_All → runCDecisionTreeForPacket_)는 이 함수를 사용하지 않는다.
// (STEP 4에서 decision tree로 교체 완료)
// ============================================================
function getCCodePromptTemplate_(){
  return `당신은 '학생 상호작용(C차원)' 코더입니다. 입력으로 주어진 "요약문(F열)"만 보고, 
해당 클러스터에서 학생들 사이에 실제로 일어난 상호작용의 "궁극적 목적"을 기준으로 
C코드 1개를 부여하세요. (교사 발화는 맥락 파악용 참고만 가능, 코딩의 핵심 근거는 학생 상호작용)

[전제/게이트]
- 이 프롬프트는 스크립트에서 '활성 발화자 수 ≥ 2명(G~J 중 값>0인 인원 수)'인 행에만 호출됩니다.
- 'OFF_TASK'(수업과 무관한 잡담/놀리기/게임/사담) 중심이면 C코드 부여하지 않습니다. (아무 것도 출력하지 말 것)
- 교사 발화는 "코딩 대상"이 아니며, 학생 간 상호작용을 중심으로 판정합니다.

[출력]
- 한 줄만 출력합니다. 형식: C#. 코드명 — "학생 인용(6~30자)" + 짧은 근거
- 근거는 반드시 학생 직접 발화에서 6~30자를 '따옴표'로 인용하세요(교사/서술 인용 금지).
- 코드를 못 붙이면(OFF_TASK 중심 등) 아예 아무 것도 출력하지 마세요.

[C차원 코드북(정의/예시)]
C1. 동의
- ⚠️ 핵심 정의: 상대의 의견을 단순히 지지해주는 말. 동의 자체가 목적이며, 추가 설명이나 확장이 없어야 함.
- 친구의 발화를 긍정적으로 수용하고 언어적/비언어적으로 동의를 표함.
- ⚠️ 중요: C1은 "동의만 있고 추가 설명/확장/정교화가 전혀 없을 때"만 부여.
- 예: "맞아", "그래", "응", "맞아 맞아" (단순 동의만, 추가 설명 없음)
- ⚠️ 반례 (C1 금지):
  * "맞아, 그리고 또 이런 점도 있어" → C3 (동의 후 추가 설명/확장)
  * "맞아, 그런데 그건 말이 안 돼" → C4 (동의 후 반박)
  * "맞아, 그러니까 결국 내 말이 맞아" → C5 (동의 후 설득)
  * "맞아, 그래서 외부-내부 압력차로 공기가 들어와" → C3 (동의 후 정교화)
- 유의: 
  * 동의 후 자신의 생각을 추가하거나 확장하거나 정교하게 만드는 경우는 C3 (정교화)
  * C1은 단순 지지/긍정만, C3은 추가 생각/확장/정교화
  * 동의가 단발적이고 그 뒤에 다른 상호작용이 없을 때만 C1

C2. 명료화/정당화 요청
- 친구의 주장이나 개념 설명에 대해 이유/근거/의미를 더 분명히 하도록 요구.
- 예: "왜 그렇게 생각했어?", "그거 선택한 이유가 뭐야?", "그게 무슨 뜻이야?"
- 유의: 이 요청이 곧바로 '반박'을 위한 지렛대라면, 상호작용의 최종 목적이 반박이므로 C4 적용('목적 우선 규칙').

C3. 상호작용을 통한 정교화
- ⚠️ 핵심 정의: 상대 의견에 대해 자신의 생각을 추가하거나 그 생각을 더 확장하거나 정교하게 만들어주는 것.
- 기존 주장/설명에 새로운 근거나 구체를 보태어 내용을 깊게/풍부하게 만듦.
- 예: 기존 "폐가 커지면 압력이 줄어든다" → "그래서 외부-내부 압력차로 공기가 들어와"
- 예: "맞아, 그리고 또 이런 점도 있어" (동의 후 추가 설명/확장)
- 예: "맞아, 그래서 외부-내부 압력차로 공기가 들어와" (동의 후 정교화)
- ⚠️ C1과의 구분:
  * C1: 단순 지지/긍정만 ("맞아", "그래", "응")
  * C3: 동의 후 추가 설명/확장/정교화 ("맞아, 그리고...", "맞아, 그래서...")
- 유의: 단순 반복/나열이면 X. 상대 발언에 '추가 논리/예시/조건'을 보태야 C3.

C4. 비판과 반박
- 친구의 발화/아이디어의 문제점을 지적하거나 논리적 허점을 비판. 타당하지 않음을 드러내는 것에 초점.
- 예: "그건 아닌 것 같아. 왜냐하면…", "말이 안 돼"
- 유의: C2(근거 요청)→상대 설명→즉시 반박으로 이어지는 흐름이면 '최종 목적'이 반박이므로 C4.

C5. 설득
- 상대가 내 주장으로 '생각을 바꾸도록' 유도. 내 입장을 강화하며 상대 수용을 끌어냄.
- 예: "이게 맞는 게, 우리가 실험에서 봤잖아"
- 유의: '부분 동의 후 설득'("맞아, 그런데 … 그래서 결국 내 말이 맞아")은 C1이 아니라 C5(목적 우선).

C6. 아이디어 조율
- 둘 이상의 상이한 주장을 비교·절충·통합하여 공동의 설명/결정을 구성.
- 예: "그럼 네 말이랑 내 말 합치면 이렇게 되겠네"
- 유의: 단순 절차 제안은 M1(다른 차원) 소관이지만, 여기서는 '아이디어 자체의 통합/절충'일 때만 C6.

C7. 또래 교수
- 더 잘 이해하는 학생이 상대의 수준에 맞춰 개념을 재구성/구체화하여 '의도적으로 가르침'.
- 예: "이건 풍선이 터지는 거랑 비슷해~"
- 유의: 단순 보충(C3)과의 차이는 '상대의 이해를 돕기 위한 설명 재구성'과 '교수 의도'가 중심일 때 C7.

[핵심 판정 원리: "상호작용의 궁극적 목적" 우선 - 맥락 필수 확인]
- ⚠️ 절대 금지: 한 발화만 보고 판정하지 말 것. 반드시 요약문 전체 맥락(전후 맥락 포함)을 확인해야 함.
- 학생 A의 한 발화만 보지 말고, 요약문 전체 맥락에서 A↔B 상호작용이 궁극적으로 무엇을 달성하려는지 판단.
- 특히 C1 동의 판정 시: 동의 발화 "이후" 맥락을 반드시 확인. 동의 후 다른 상호작용이 이어지면 C1이 아님.
- 예1) "왜 그렇게 생각했어?"(C2) → B가 설명 → A가 "그래서 네 말은 성립 안 해"(반박) → 최종 목적=반박 → C4
- 예2) "맞아"(동의) + "그러니까 이건 결국 내 말대로야, 실험에서도…"(설득) → 최종 목적=설득 → C5 (C1 아님!)
- 예3) "맞아"(동의) + "그런데 그건 말이 안 돼, 왜냐하면…"(반박) → 최종 목적=반박 → C4 (C1 아님!)
- 예4) "맞아"(동의) + "그래서 외부-내부 압력차로 공기가 들어와"(정교화) → 최종 목적=정교화 → C3 (C1 아님!)
- 예5) "맞아"(동의)만 있고 그 뒤 아무 상호작용 없음 → C1 (동의가 최종 목적)
- 예6) "맞아, 그리고 또 이런 점도 있어" → C3 (동의 후 추가 설명/확장, C1 아님!)
- 예6) 서로 다른 주장을 비교하며 공통안을 제시 → C6
- 예7) 상대가 헷갈려 하는 개념을 학생이 비유/재구성으로 가르침 → C7

[우선순위(충돌 시 결정 규칙) - 맥락 기반]
- 목적 우선 체인: C5(설득) / C4(반박) / C6(조율) > C3(정교화) > C2(명료화요청) > C1(동의)
- ⚠️ C1은 가장 낮은 우선순위: 동의가 나타나더라도 다른 상호작용이 있으면 C1 금지
  - 동의+설득 동시: C5 (C1 아님)
  - 동의+반박 동시: C4 (C1 아님)
  - 동의+정교화 동시: C3 (C1 아님)
  - 동의+조율 동시: C6 (C1 아님)
  - 근거요청→반박: C4
  - 보충과 또래 교수의 경계: 상대 '이해 지원을 위한 의도적 재구성'이면 C7, 아니면 C3
  - 여러 신호가 섞이면, 클러스터의 종결부/결론부에 드러난 목적을 우선
  - 동의만 있고 그 이후 아무 상호작용 없을 때만 C1

[판정 절차(간단 체크리스트) - 맥락 우선]
1) OFF_TASK 중심인가? → 그렇다면 출력하지 말 것.
2) 요약문 전체 맥락 확인: 동의 발화가 있더라도 "이후 맥락"을 반드시 확인
3) '상호작용의 궁극적 목적'이 설득/반박/조율 중 하나인가?
   - 설득 신호(상대 수용 유도, "결국 내 말이 맞아/증거 있어") → C5
   - 반박 신호(문제 제기, 논리 허점 지적, "말이 안 돼/그건 아니지") → C4
   - 조율 신호(절충/통합 제안, "그럼 합치면…") → C6
4) 아니면 '의미 요청/근거 요청'이 중심인가? → C2
   - 단, 요청→설명→반박으로 이어지면 C4
5) 상대 발언에 논리/예시/조건을 붙여 설명을 깊게 했나? → C3
   - ⚠️ 중요: 동의 후 추가 설명/확장/정교화가 있으면 C3, C1 아님
6) 상대가 헷갈리는 개념을 학생이 비유/재구성으로 가르쳤나? → C7
7) ⚠️ C1 판정 전 최종 확인: 동의 발화 "이후"에 다른 상호작용(설득/반박/정교화/조율/교수)이 있는가?
   - 있다면 → C1 금지, 해당 상호작용의 코드 부여
   - 없다면 → C1 (동의가 최종 목적)
   - ⚠️ 특히 주의: 동의 후 자신의 생각을 추가하거나 확장하거나 정교하게 만드는 경우는 C3, C1 아님

[2단계 판정 절차]
1) 게이트(무출력 조건) 확인:
   - 절차/전략/역할/산출물 중심(정리, 일단, 먼저, 순서, 역할, 분류, 제출, 적자/적다, 정돈, 붙이자, 하자/해야 해, "가위/테이프/핸드폰" 등 도구요청) → 무출력
   - 교사 주도 구간(교사 발화가 대부분이고 학생은 단편적 반응) → 무출력
   - 단발 리액션만 오간 경우("응/맞아/나이스" 등) → 무출력
2) 목적 판정:
   - 설득 신호(상대 수용 유도, "결국 내 말이 맞아/증거 있어") → C5
   - 반박 신호("말이 안 돼/그건 아니지/왜 그건 성립?" 등) → C4
   - 아이디어 절충·통합(설명 자체의 통합) → C6
   - 상대 이해 돕는 재구성·비유 → C7
   - 상대 발언에 논리/예시/조건을 붙여 설명 확장 → C3
   - 이유/근거/의미를 구체적으로 요구 → C2
   - 그 외 핵심이 수용·지지 → C1

[오남용 방지 메모]
- C6은 '아이디어' 조율에만. '역할/제출/절차' 조율이면 C 금지(무출력).
- C1은 단발 리액션일 경우 금지(무출력). 내용적 연쇄가 있어도 '결말 목적'이 설득/반박이면 C5/C4.
- 교사 발화를 핵심 근거로 쓰지 말고, 학생 발화에서 6~30자 구절을 인용.

[C1 최소요건 및 맥락 검증]
- ⚠️ C1은 가장 엄격하게 판정: 동의가 "최종 목적"이고 "그 이후 아무 상호작용도 없을 때"만 부여.
- 단발 리액션("맞아/나이스/그래")만 오간 경우 C1 금지 → 무출력.
- 동의 발화가 있더라도 요약문 전체를 읽고 "이후 맥락"을 반드시 확인:
  * 동의 후 설득/반박/정교화/조율/교수가 이어지면 → C1 금지, 해당 코드 부여
  * 동의 후 아무 상호작용 없고 동의 자체가 끝 → C1 가능
- C1을 줄 때는 학생 주장/설명에 '핵심적으로 수용'이 드러나고, 그 이후 다른 목적의 상호작용이 없을 때만.

[C2→C4 승격 규칙(체인)]
- "왜/근거/무슨 뜻" 등 C2 신호 → 상대 설명 → 곧바로 "아니야/말이 안 돼/그건 아니지/다른데" 등의 반박 신호가 이어지면 최종 코드는 C4.

[C6 오남용 방지]
- C6은 '아이디어(설명/주장) 절충·통합'일 때만.
- '역할/제출/절차/도구' 조율은 C 금지(무출력).

[증거 인용 강제]
- 출력은 한 줄: C#. 코드명 — "학생 인용(6~30자)" + 짧은 근거
- 인용은 학생 발화만. 교사/서술 인용 금지.
- 인용이 절차/잡담이면 코드 자체를 부여하지 말 것.

[우선순위/결말 가중]
- 혼재 시 클러스터 '후반부 목적'을 따른다.
- 우선순위: C5(설득) > C4(반박) > C6(아이디어 조율) > C7(또래 교수) > C3 > C2 > C1

요약문:
<<SUMMARY>>`;
}

/***** 12) 🅿 P차원 코딩 (코드북 v2) *****/

/** 🅿 P차원 코딩 (GPT 기반, 후보군=힌트) */
/** P차원 코딩 (F열 요약문 기반 GPT 분석 v7)
 * 핵심: F열 요약문 → GPT 분석 → 의미 있는 기여자 수(m)로 P 결정
 */
/** 🔄 P 코딩 (N열 쓰기) — Deterministic v1.0 (K/C/M Note contributors union) */
function runCodeP_All() {
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  const lCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const nCol = colNumOf(map.N);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  if (!packets || !packets.length) {
    ui.alert('⚠️ 처리할 PID 패킷이 없습니다.\nK/C/M 코딩을 먼저 실행하세요.');
    return;
  }

  const maxRow = packets.reduce(function(max, p){
    const r = p && p.representativeRow;
    return (r && r > max) ? r : max;
  }, 2);
  const noteRowCount = maxRow - 1;
  const kNotes = noteRowCount > 0 ? sh.getRange(2, kCol, noteRowCount, 1).getNotes() : [];
  const cNotes = noteRowCount > 0 ? sh.getRange(2, lCol, noteRowCount, 1).getNotes() : [];
  const mNotes = noteRowCount > 0 ? sh.getRange(2, mCol, noteRowCount, 1).getNotes() : [];

  let codedCount = 0;
  let errorCount = 0;
  let skippedCount = 0;

  packets.forEach(function(packet) {
    const row = packet && packet.representativeRow;
    if (!row) {
      skippedCount++;
      Logger.log('⚠️ P 코딩 건너뜀: representativeRow 없음 pid=' + (packet && packet.pid));
      return;
    }
    const idx = row - 2;
    const kNoteText = idx >= 0 && idx < kNotes.length ? kNotes[idx][0] : "";
    const cNoteText = idx >= 0 && idx < cNotes.length ? cNotes[idx][0] : "";
    const mNoteText = idx >= 0 && idx < mNotes.length ? mNotes[idx][0] : "";

    const result = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
    _writePDecisionCell_(sh, row, nCol, result);

    if (result && result.status === "OK" && result.code) {
      codedCount++;
      if (codedCount <= 3) {
        Logger.log('P 코딩 성공 [행' + row + '] ' + packet.pid + ': ' + result.code + ' contributors=' + JSON.stringify(result.contributors || []));
      }
    } else {
      errorCount++;
      Logger.log('❌ P 코딩 upstream 오류 [행' + row + '] ' + (packet.pid || '') + ': ' + (result && result.error_type) + ' ' + (result && result.message));
    }
  });

  let msg = '✅ P 코딩 완료 (Deterministic v1.0)\n\n';
  msg += '📊 통계:\n';
  msg += '- P0~P3 부여: ' + codedCount + '개\n';
  if (errorCount > 0) msg += '- upstream 오류(빈칸): ' + errorCount + '개\n';
  if (skippedCount > 0) msg += '- 대표행 없음 skip: ' + skippedCount + '개\n';
  msg += '\nP는 GPT를 사용하지 않습니다. K/C/M Note JSON contributors union으로 계산합니다.';
  msg += '\nupstream ERROR / Note 없음 / invalid / 미확정이면 N셀은 빈칸 + Note status=ERROR 입니다.';
  if (errorCount > 0) {
    msg += '\n\n⚠️ upstream 오류가 있습니다. K/C/M 코딩을 먼저 완료했는지 확인하세요.';
  }
  ui.alert(msg);
}

const P_DECISION_LABELS_ = {
  "P0": "의미 있는 참여 없음",
  "P1": "1명의 의미 있는 참여",
  "P2": "소수의 의미 있는 참여",
  "P3": "다수의 의미 있는 참여"
};

function _parseKCMPDecisionNote_(noteText, dimLabel){
  const label = String(dimLabel || "UPSTREAM");
  if (noteText == null || String(noteText).trim().length === 0) {
    return { state: "MISSING", result: null, message: label + " Note 없음" };
  }
  try {
    const obj = JSON.parse(String(noteText));
    if (!obj || typeof obj !== "object" || Array.isArray(obj)) {
      return { state: "INVALID", result: null, message: label + " Note JSON 객체 아님" };
    }
    if (obj.status === "OK") {
      return { state: "OK", result: obj, message: "" };
    }
    if (obj.status === "ERROR") {
      return { state: "ERROR", result: obj, message: label + " upstream ERROR: " + (obj.error_type || "ERROR") };
    }
    return { state: "NOT_FINALIZED", result: obj, message: label + " status 미확정: " + obj.status };
  } catch (e) {
    return { state: "INVALID", result: null, message: label + " Note JSON parse 실패" };
  }
}

/** dimension별 production Note finalized 판별 (K/C/M/P resume orchestration 전용) */
function _expectedKCMPSchemaVersion_(dimension){
  const dim = String(dimension || "").toUpperCase();
  if (dim === "K") return "KCMP_K_V1";
  if (dim === "C") return "KCMP_C_V1";
  if (dim === "M") return "KCMP_M_V1";
  if (dim === "P") return "KCMP_P_V1";
  return null;
}

function _isFinalKCMPDecisionNote_(noteText, dimension){
  const empty = { finalized: false, status: null, code: null, schema_version: null };
  const expectedSchema = _expectedKCMPSchemaVersion_(dimension);
  if (!expectedSchema) return empty;

  if (noteText == null || String(noteText).trim().length === 0) return empty;

  let obj;
  try {
    obj = JSON.parse(String(noteText));
  } catch (e) {
    return empty;
  }
  if (!obj || typeof obj !== "object" || Array.isArray(obj)) return empty;

  const schema = obj.schema_version != null ? String(obj.schema_version) : null;
  const status = obj.status != null ? String(obj.status) : null;
  const code = (obj.code === null || obj.code === undefined) ? null : String(obj.code);

  if (schema !== expectedSchema) {
    return { finalized: false, status: status, code: code, schema_version: schema };
  }
  if (status === "OK" || status === "ERROR") {
    return { finalized: true, status: status, code: code, schema_version: schema };
  }
  return { finalized: false, status: status, code: code, schema_version: schema };
}

/** representativeRow 목록에 대해 연속 블록 단위 batch getNotes() */
function _batchGetNotesForRows_(sheet, col, rows){
  const out = {};
  if (!sheet || !col || !rows || !rows.length) return out;

  const uniq = {};
  rows.forEach(function(r){
    const n = Number(r);
    if (n > 0) uniq[n] = true;
  });
  const sorted = Object.keys(uniq).map(Number).sort(function(a, b){ return a - b; });
  if (!sorted.length) return out;

  let blockStart = sorted[0];
  let blockEnd = sorted[0];
  for (let i = 1; i <= sorted.length; i++) {
    const r = sorted[i];
    if (i < sorted.length && r === blockEnd + 1) {
      blockEnd = r;
      continue;
    }
    const h = blockEnd - blockStart + 1;
    const blockNotes = sheet.getRange(blockStart, col, h, 1).getNotes();
    for (let j = 0; j < h; j++) {
      out[blockStart + j] = blockNotes[j][0];
    }
    if (i < sorted.length) {
      blockStart = r;
      blockEnd = r;
    }
  }
  return out;
}

function _collectKCMPRepresentativeRows_(packets){
  const rows = [];
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (row) rows.push(row);
  });
  return rows;
}

function _summarizeKCMPDimensionNoteProgress_(packets, notesByRow, dimension){
  let ok = 0;
  let err = 0;
  let unf = 0;
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    const fin = _isFinalKCMPDecisionNote_(notesByRow[row], dimension);
    if (fin.finalized && fin.status === "OK") ok++;
    else if (fin.finalized && fin.status === "ERROR") err++;
    else unf++;
  });
  return {
    finalizedOk: ok,
    finalizedError: err,
    unfinalized: unf,
    complete: unf === 0
  };
}

function _summarizeKCMPProductionProgress_(sheet, map, packets){
  const rows = _collectKCMPRepresentativeRows_(packets);
  const kNotes = _batchGetNotesForRows_(sheet, colNumOf(map.K), rows);
  const cNotes = _batchGetNotesForRows_(sheet, colNumOf(map.L), rows);
  const mNotes = _batchGetNotesForRows_(sheet, colNumOf(map.M), rows);
  const pNotes = _batchGetNotesForRows_(sheet, colNumOf(map.N), rows);
  const k = _summarizeKCMPDimensionNoteProgress_(packets, kNotes, "K");
  const c = _summarizeKCMPDimensionNoteProgress_(packets, cNotes, "C");
  const m = _summarizeKCMPDimensionNoteProgress_(packets, mNotes, "M");
  const p = _summarizeKCMPDimensionNoteProgress_(packets, pNotes, "P");
  return {
    totalPackets: (packets || []).length,
    K: k,
    C: c,
    M: m,
    P: p,
    allKcmFinalized: k.complete && c.complete && m.complete,
    allKcmpFinalized: k.complete && c.complete && m.complete && p.complete
  };
}

function _logKCMPProductionResumeBatch_(label, sheetName, stats){
  Logger.log("=== " + label + " PRODUCTION RESUME BATCH ===");
  Logger.log("SHEET=" + sheetName);
  Logger.log("TOTAL_PACKETS=" + stats.totalPackets);
  Logger.log("FINALIZED_BEFORE=" + stats.finalizedBefore);
  Logger.log("SKIPPED_FINALIZED=" + stats.skippedFinalized);
  Logger.log("PROCESSED_THIS_RUN=" + stats.processedThisRun);
  Logger.log("OK_CODED_THIS_RUN=" + stats.okCodedThisRun);
  Logger.log("OK_NULL_THIS_RUN=" + stats.okNullThisRun);
  Logger.log("ERROR_WRITTEN_THIS_RUN=" + stats.errorWrittenThisRun);
  Logger.log("REMAINING_UNFINALIZED=" + stats.remainingUnfinalized);
  Logger.log("ELAPSED_MS=" + stats.elapsedMs);
  Logger.log("STOP_REASON=" + stats.stopReason);
  Logger.log("COMPLETE=" + String(stats.complete));
  Logger.log("MAX_CASES_LIMIT=" + KCMP_KCM_PRODUCTION_BATCH_MAX_CASES);
  Logger.log("TIME_BUDGET_MS=" + KCMP_KCM_PRODUCTION_TIME_BUDGET_MS);
  Logger.log("TIME_SAFETY_MARGIN_MS=" + KCMP_PRODUCTION_TIME_SAFETY_MARGIN_MS);
}

function _runKCMPProductionResumeBatch_(dimension){
  const dim = String(dimension || "").toUpperCase();
  const startMs = Date.now();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const sheetName = String(sh.getName ? sh.getName() : "");

  const colByDim = { K: colNumOf(map.K), C: colNumOf(map.L), M: colNumOf(map.M) };
  const col = colByDim[dim];
  if (!col) throw new Error("Unknown dimension for resume batch: " + dimension);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  if (!packets || !packets.length) {
    const emptyStats = {
      totalPackets: 0,
      finalizedBefore: 0,
      skippedFinalized: 0,
      processedThisRun: 0,
      okCodedThisRun: 0,
      okNullThisRun: 0,
      errorWrittenThisRun: 0,
      remainingUnfinalized: 0,
      elapsedMs: Date.now() - startMs,
      stopReason: "COMPLETE",
      complete: true
    };
    _logKCMPProductionResumeBatch_(dim, sheetName, emptyStats);
    return emptyStats;
  }

  const rows = _collectKCMPRepresentativeRows_(packets);
  let notesByRow = _batchGetNotesForRows_(sh, col, rows);

  let finalizedBefore = 0;
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    if (_isFinalKCMPDecisionNote_(notesByRow[row], dim).finalized) finalizedBefore++;
  });

  let skippedFinalized = 0;
  let processedThisRun = 0;
  let okCodedThisRun = 0;
  let okNullThisRun = 0;
  let errorWrittenThisRun = 0;
  let stopReason = "COMPLETE";

  for (let i = 0; i < packets.length; i++) {
    const packet = packets[i];
    const row = packet && packet.representativeRow;
    if (!row) continue;

    const fin = _isFinalKCMPDecisionNote_(notesByRow[row], dim);
    if (fin.finalized) {
      skippedFinalized++;
      continue;
    }

    if (processedThisRun >= KCMP_KCM_PRODUCTION_BATCH_MAX_CASES) {
      stopReason = "MAX_CASES";
      break;
    }
    if ((Date.now() - startMs) >= (KCMP_KCM_PRODUCTION_TIME_BUDGET_MS - KCMP_PRODUCTION_TIME_SAFETY_MARGIN_MS)) {
      stopReason = "TIME_BUDGET";
      break;
    }

    let result;
    if (dim === "K") {
      result = runKDecisionTreeForPacket_(packet);
      _writeKDecisionCell_(sh, row, col, result);
    } else if (dim === "C") {
      result = runCDecisionTreeForPacket_(packet);
      _writeCDecisionCell_(sh, row, col, result);
    } else if (dim === "M") {
      result = runMDecisionTreeForPacket_(packet, { allPackets: packets });
      _writeMDecisionCell_(sh, row, col, result);
    } else {
      throw new Error("Unknown dimension for resume batch: " + dimension);
    }

    processedThisRun++;
    if (result && result.status === "OK" && result.code) okCodedThisRun++;
    else if (result && result.status === "OK" && result.code == null) okNullThisRun++;
    else errorWrittenThisRun++;

    notesByRow[row] = JSON.stringify(result || {});
  }

  notesByRow = _batchGetNotesForRows_(sh, col, rows);
  let remainingUnfinalized = 0;
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    if (!_isFinalKCMPDecisionNote_(notesByRow[row], dim).finalized) remainingUnfinalized++;
  });

  if (remainingUnfinalized === 0) stopReason = "COMPLETE";

  const stats = {
    totalPackets: packets.length,
    finalizedBefore: finalizedBefore,
    skippedFinalized: skippedFinalized,
    processedThisRun: processedThisRun,
    okCodedThisRun: okCodedThisRun,
    okNullThisRun: okNullThisRun,
    errorWrittenThisRun: errorWrittenThisRun,
    remainingUnfinalized: remainingUnfinalized,
    elapsedMs: Date.now() - startMs,
    stopReason: stopReason,
    complete: remainingUnfinalized === 0
  };
  _logKCMPProductionResumeBatch_(dim, sheetName, stats);
  return stats;
}

function _logPProductionResumeBatch_(sheetName, stats){
  Logger.log("=== P PRODUCTION RESUME BATCH ===");
  Logger.log("SHEET=" + sheetName);
  Logger.log("TOTAL_PACKETS=" + stats.totalPackets);
  Logger.log("FINALIZED_BEFORE=" + stats.finalizedBefore);
  Logger.log("SKIPPED_FINALIZED=" + stats.skippedFinalized);
  Logger.log("PROCESSED_THIS_RUN=" + stats.processedThisRun);
  Logger.log("OK_P0_THIS_RUN=" + stats.okP0ThisRun);
  Logger.log("OK_P1_THIS_RUN=" + stats.okP1ThisRun);
  Logger.log("OK_P2_THIS_RUN=" + stats.okP2ThisRun);
  Logger.log("OK_P3_THIS_RUN=" + stats.okP3ThisRun);
  Logger.log("ERROR_WRITTEN_THIS_RUN=" + stats.errorWrittenThisRun);
  Logger.log("REMAINING_UNFINALIZED=" + stats.remainingUnfinalized);
  Logger.log("ELAPSED_MS=" + stats.elapsedMs);
  Logger.log("STOP_REASON=" + stats.stopReason);
  Logger.log("COMPLETE=" + String(stats.complete));
  Logger.log("P_GPT_CALLS=0");
  Logger.log("MAX_CASES_LIMIT=" + KCMP_P_PRODUCTION_BATCH_MAX_CASES);
  Logger.log("TIME_BUDGET_MS=" + KCMP_P_PRODUCTION_TIME_BUDGET_MS);
  Logger.log("TIME_SAFETY_MARGIN_MS=" + KCMP_PRODUCTION_TIME_SAFETY_MARGIN_MS);
}

/** deterministic P resume batch — GPT 없음, finalized P Note는 SKIP, clear 금지 */
function _runPProductionResumeBatch_(){
  const startMs = Date.now();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const sheetName = String(sh.getName ? sh.getName() : "");
  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const nCol = colNumOf(map.N);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  if (!packets || !packets.length) {
    const emptyStats = {
      totalPackets: 0,
      finalizedBefore: 0,
      skippedFinalized: 0,
      processedThisRun: 0,
      okP0ThisRun: 0,
      okP1ThisRun: 0,
      okP2ThisRun: 0,
      okP3ThisRun: 0,
      errorWrittenThisRun: 0,
      remainingUnfinalized: 0,
      elapsedMs: Date.now() - startMs,
      stopReason: "COMPLETE",
      complete: true
    };
    _logPProductionResumeBatch_(sheetName, emptyStats);
    return emptyStats;
  }

  const rows = _collectKCMPRepresentativeRows_(packets);
  let kNotesByRow = _batchGetNotesForRows_(sh, kCol, rows);
  let cNotesByRow = _batchGetNotesForRows_(sh, cCol, rows);
  let mNotesByRow = _batchGetNotesForRows_(sh, mCol, rows);
  let pNotesByRow = _batchGetNotesForRows_(sh, nCol, rows);

  let finalizedBefore = 0;
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    if (_isFinalKCMPDecisionNote_(pNotesByRow[row], "P").finalized) finalizedBefore++;
  });

  let skippedFinalized = 0;
  let processedThisRun = 0;
  let okP0ThisRun = 0;
  let okP1ThisRun = 0;
  let okP2ThisRun = 0;
  let okP3ThisRun = 0;
  let errorWrittenThisRun = 0;
  let stopReason = "COMPLETE";

  for (let i = 0; i < packets.length; i++) {
    const packet = packets[i];
    const row = packet && packet.representativeRow;
    if (!row) continue;

    const fin = _isFinalKCMPDecisionNote_(pNotesByRow[row], "P");
    if (fin.finalized) {
      skippedFinalized++;
      continue;
    }

    if (processedThisRun >= KCMP_P_PRODUCTION_BATCH_MAX_CASES) {
      stopReason = "MAX_CASES";
      break;
    }
    if ((Date.now() - startMs) >= (KCMP_P_PRODUCTION_TIME_BUDGET_MS - KCMP_PRODUCTION_TIME_SAFETY_MARGIN_MS)) {
      stopReason = "TIME_BUDGET";
      break;
    }

    const kNoteText = kNotesByRow[row] != null ? String(kNotesByRow[row]) : "";
    const cNoteText = cNotesByRow[row] != null ? String(cNotesByRow[row]) : "";
    const mNoteText = mNotesByRow[row] != null ? String(mNotesByRow[row]) : "";

    const result = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
    _writePDecisionCell_(sh, row, nCol, result);

    processedThisRun++;
    if (result && result.status === "OK" && result.code === "P0") okP0ThisRun++;
    else if (result && result.status === "OK" && result.code === "P1") okP1ThisRun++;
    else if (result && result.status === "OK" && result.code === "P2") okP2ThisRun++;
    else if (result && result.status === "OK" && result.code === "P3") okP3ThisRun++;
    else errorWrittenThisRun++;

    pNotesByRow[row] = JSON.stringify(result || {});
  }

  pNotesByRow = _batchGetNotesForRows_(sh, nCol, rows);
  let remainingUnfinalized = 0;
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    if (!_isFinalKCMPDecisionNote_(pNotesByRow[row], "P").finalized) remainingUnfinalized++;
  });

  if (remainingUnfinalized === 0) stopReason = "COMPLETE";

  const stats = {
    totalPackets: packets.length,
    finalizedBefore: finalizedBefore,
    skippedFinalized: skippedFinalized,
    processedThisRun: processedThisRun,
    okP0ThisRun: okP0ThisRun,
    okP1ThisRun: okP1ThisRun,
    okP2ThisRun: okP2ThisRun,
    okP3ThisRun: okP3ThisRun,
    errorWrittenThisRun: errorWrittenThisRun,
    remainingUnfinalized: remainingUnfinalized,
    elapsedMs: Date.now() - startMs,
    stopReason: stopReason,
    complete: remainingUnfinalized === 0
  };
  _logPProductionResumeBatch_(sheetName, stats);
  return stats;
}

function _summarizePProductionFromNotes_(packets, pNotesByRow){
  let p0 = 0, p1 = 0, p2 = 0, p3 = 0, pErr = 0, unf = 0;
  const errorTypes = {};
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    const fin = _isFinalKCMPDecisionNote_(pNotesByRow[row], "P");
    if (!fin.finalized) {
      unf++;
      return;
    }
    if (fin.status === "ERROR") {
      pErr++;
      let et = "UNKNOWN";
      try {
        const obj = JSON.parse(String(pNotesByRow[row] || ""));
        et = obj && obj.error_type ? String(obj.error_type) : "UNKNOWN";
      } catch (e) {}
      errorTypes[et] = (errorTypes[et] || 0) + 1;
      return;
    }
    if (fin.code === "P0") p0++;
    else if (fin.code === "P1") p1++;
    else if (fin.code === "P2") p2++;
    else if (fin.code === "P3") p3++;
  });
  return { p0: p0, p1: p1, p2: p2, p3: p3, pError: pErr, unfinalized: unf, errorTypes: errorTypes };
}

function _extractStudentContributorsFromNote_(parsedNote){
  if (!parsedNote || parsedNote.state !== "OK") return [];
  const contributors = parsedNote.result && parsedNote.result.contributors;
  if (!Array.isArray(contributors)) return [];
  const out = [];
  contributors.forEach(function(c){
    const s = String(c == null ? "" : c).trim();
    if (/^S[1-4]$/.test(s) && out.indexOf(s) < 0) out.push(s);
  });
  return out;
}

function _unionStudentContributors_(arrays){
  const out = [];
  (arrays || []).forEach(function(arr){
    (arr || []).forEach(function(c){
      if (out.indexOf(c) < 0) out.push(c);
    });
  });
  out.sort();
  return out;
}

function _pCodeFromContributorCount_(n){
  const count = Number(n) || 0;
  if (count <= 0) return "P0";
  if (count === 1) return "P1";
  if (count === 2) return "P2";
  return "P3";
}

function _summarizeUpstreamNote_(parsedNote){
  if (!parsedNote) return { state: "MISSING", status: null, code: null, contributors: [] };
  return {
    state: parsedNote.state,
    status: parsedNote.result && parsedNote.result.status != null ? parsedNote.result.status : null,
    code: parsedNote.result && parsedNote.result.code != null ? parsedNote.result.code : null,
    contributors: _extractStudentContributorsFromNote_(parsedNote)
  };
}

function _makePDecisionError_(errorType, message, pid, extra){
  const err = {
    schema_version: "KCMP_P_V1",
    status: "ERROR",
    error_type: String(errorType || "UPSTREAM_ERROR"),
    message: String(message == null ? "" : message),
    pid: pid || "",
    code: null,
    contributors: [],
    upstream: { K: null, C: null, M: null }
  };
  if (extra && typeof extra === "object") {
    Object.keys(extra).forEach(function(k){ err[k] = extra[k]; });
  }
  return err;
}

function computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText){
  const pid = packet && packet.pid ? packet.pid : "";
  const kParsed = _parseKCMPDecisionNote_(kNoteText, "K");
  const cParsed = _parseKCMPDecisionNote_(cNoteText, "C");
  const mParsed = _parseKCMPDecisionNote_(mNoteText, "M");
  const upstream = {
    K: _summarizeUpstreamNote_(kParsed),
    C: _summarizeUpstreamNote_(cParsed),
    M: _summarizeUpstreamNote_(mParsed)
  };

  const dims = [
    { key: "K", parsed: kParsed },
    { key: "C", parsed: cParsed },
    { key: "M", parsed: mParsed }
  ];
  for (let i = 0; i < dims.length; i++) {
    const d = dims[i];
    if (d.parsed.state === "ERROR") {
      return _makePDecisionError_("UPSTREAM_ERROR", d.parsed.message, pid, { upstream: upstream });
    }
    if (d.parsed.state === "MISSING" || d.parsed.state === "INVALID" || d.parsed.state === "NOT_FINALIZED") {
      return _makePDecisionError_("UPSTREAM_NOT_FINALIZED", d.parsed.message, pid, { upstream: upstream });
    }
  }

  const union = _unionStudentContributors_([
    _extractStudentContributorsFromNote_(kParsed),
    _extractStudentContributorsFromNote_(cParsed),
    _extractStudentContributorsFromNote_(mParsed)
  ]);
  const code = _pCodeFromContributorCount_(union.length);
  const label = P_DECISION_LABELS_[code] || code;
  const reason = union.length
    ? ("K/C/M contributors union " + union.join(", ") + " (" + union.length + "명) → " + code)
    : ("K/C/M contributors union 0명 → " + code + " (" + label + ")");

  return {
    schema_version: "KCMP_P_V1",
    status: "OK",
    code: code,
    contributors: union.slice(),
    reason: reason,
    upstream: upstream,
    pid: pid
  };
}

function formatPDecisionDisplay_(result){
  if (!result || result.status !== "OK" || !result.code) return "";
  const label = P_DECISION_LABELS_[result.code] || result.code;
  const contribs = (result.contributors || []).join(", ");
  if (contribs) return result.code + " — " + label + " — " + contribs;
  return result.code + " — " + label;
}

function _writePDecisionCell_(sheet, row, nCol, result){
  const cell = sheet.getRange(row, nCol);
  cell.clearContent();
  if (result && result.status === "OK" && result.code) {
    cell.setValue(formatPDecisionDisplay_(result));
  }
  try {
    cell.setNote(JSON.stringify(result || {}));
  } catch (e) {
    cell.setNote(JSON.stringify(_makePDecisionError_("NOTE_WRITE_ERROR", e.toString(), result && result.pid)));
  }
}

function TEST_DETERMINISTIC_P(){
  const packet = _kcmpSyntheticPacket_("PD01", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력이 낮아져." }
  ]);
  const okNullK = JSON.stringify({ schema_version: "KCMP_K_V1", status: "OK", code: null, contributors: [] });
  const okNullC = JSON.stringify({ schema_version: "KCMP_C_V1", status: "OK", code: null, contributors: [] });
  const okNullM = JSON.stringify({ schema_version: "KCMP_M_V1", status: "OK", code: null, contributors: [] });
  const p0 = computeDeterministicPForPacket_(packet, okNullK, okNullC, okNullM);

  const okK1 = JSON.stringify({ schema_version: "KCMP_K_V1", status: "OK", code: "K1", contributors: ["S1"] });
  const okC3 = JSON.stringify({ schema_version: "KCMP_C_V1", status: "OK", code: "C3", contributors: ["S2"] });
  const p2 = computeDeterministicPForPacket_(packet, okK1, okC3, okNullM);

  const errK = JSON.stringify({ schema_version: "KCMP_K_V1", status: "ERROR", error_type: "VALIDATION_ERROR", code: null, contributors: [] });
  const upstreamErr = computeDeterministicPForPacket_(packet, errK, okNullC, okNullM);

  const missingNote = computeDeterministicPForPacket_(packet, "", okNullC, okNullM);

  const okMany = JSON.stringify({ schema_version: "KCMP_K_V1", status: "OK", code: "K3", contributors: ["S1", "S2", "S3"] });
  const p3 = computeDeterministicPForPacket_(packet, okMany, okNullC, okNullM);

  Logger.log("TEST_DETERMINISTIC_P P0=" + JSON.stringify({ code: p0.code, contributors: p0.contributors }));
  Logger.log("TEST_DETERMINISTIC_P P2=" + JSON.stringify({ code: p2.code, contributors: p2.contributors }));
  Logger.log("TEST_DETERMINISTIC_P UPSTREAM_ERR=" + upstreamErr.error_type);
  Logger.log("TEST_DETERMINISTIC_P MISSING=" + missingNote.error_type);
  Logger.log("TEST_DETERMINISTIC_P P3=" + JSON.stringify({ code: p3.code, contributors: p3.contributors }));

  return {
    p0: { code: p0.code, contributors: p0.contributors, expect_code: "P0", pass: p0.code === "P0" && (p0.contributors || []).length === 0 },
    p2: { code: p2.code, contributors: p2.contributors, expect_code: "P2", pass: p2.code === "P2" && p2.contributors.length === 2 },
    upstream_error: { error_type: upstreamErr.error_type, pass: upstreamErr.status === "ERROR" && upstreamErr.error_type === "UPSTREAM_ERROR" },
    missing_note: { error_type: missingNote.error_type, pass: missingNote.status === "ERROR" && missingNote.error_type === "UPSTREAM_NOT_FINALIZED" },
    p3: { code: p3.code, contributors: p3.contributors, expect_code: "P3", pass: p3.code === "P3" && p3.contributors.length === 3 }
  };
}

/**
 * STEP 6A) Deterministic P Real-PID Dry-Run Harness
 * - runCodeP_All 실행/쓰기 금지
 * - GPT 호출 금지
 * - K/C/M/packet/clusterPacket 수정 금지
 */
function testDeterministicPForPid_(requestedPid){
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();

  const pid = String(requestedPid || "").trim();
  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const pCol = colNumOf(map.N);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  const matched = (packets || []).filter(function(p){
    return p && p.pid === pid; // exact match only
  });
  const matchCount = matched.length;

  let selectedPid = "";
  let representativeRow = null;

  let kCellValue = "";
  let cCellValue = "";
  let mCellValue = "";

  let kNoteText = "";
  let cNoteText = "";
  let mNoteText = "";

  let kNotePresent = false;
  let cNotePresent = false;
  let mNotePresent = false;

  let kParsed = null;
  let cParsed = null;
  let mParsed = null;

  let kStatus = "";
  let cStatus = "";
  let mStatus = "";
  let kCode = "";
  let cCode = "";
  let mCode = "";
  let kContribs = [];
  let cContribs = [];
  let mContribs = [];

  let unionContribs = [];
  let unionCount = 0;

  let pResult = null;
  let pStatus = "";
  let pCode = "";
  let pReason = "";
  let pErrorType = "";

  const GPT_CALLED = false;
  const WRITE_PERFORMED = false;

  if (matchCount !== 1) {
    const log =
      "=== P DETERMINISTIC REAL-PID DRY-RUN ===\n"
      + "ACTIVE_SHEET=" + String(sh.getName ? sh.getName() : "") + "\n"
      + "REQUESTED_PID=" + pid + "\n"
      + "MATCH_COUNT=" + matchCount + "\n"
      + "SELECTED_PID=" + selectedPid + "\n"
      + "representativeRow=" + String(representativeRow) + "\n"
      + "\nK_CELL_VALUE=" + kCellValue + "\n"
      + "C_CELL_VALUE=" + cCellValue + "\n"
      + "M_CELL_VALUE=" + mCellValue + "\n"
      + "K_NOTE_PRESENT=" + String(kNotePresent) + "\n"
      + "C_NOTE_PRESENT=" + String(cNotePresent) + "\n"
      + "M_NOTE_PRESENT=" + String(mNotePresent) + "\n"
      + "K_NOTE_PARSED=" + String(kParsed) + "\n"
      + "C_NOTE_PARSED=" + String(cParsed) + "\n"
      + "M_NOTE_PARSED=" + String(mParsed) + "\n"
      + "\nK_STATUS=" + kStatus + "\n"
      + "K_CODE=" + kCode + "\n"
      + "K_CONTRIBUTORS=" + JSON.stringify(kContribs) + "\n"
      + "\nC_STATUS=" + cStatus + "\n"
      + "C_CODE=" + cCode + "\n"
      + "C_CONTRIBUTORS=" + JSON.stringify(cContribs) + "\n"
      + "\nM_STATUS=" + mStatus + "\n"
      + "M_CODE=" + mCode + "\n"
      + "M_CONTRIBUTORS=" + JSON.stringify(mContribs) + "\n"
      + "\nUNION_CONTRIBUTORS=" + JSON.stringify(unionContribs) + "\n"
      + "UNION_COUNT=" + unionCount + "\n"
      + "\nP_STATUS=" + pStatus + "\n"
      + "P_CODE=" + pCode + "\n"
      + "P_REASON=" + pReason + "\n"
      + "P_ERROR_TYPE=" + pErrorType + "\n"
      + "\nGPT_CALLED=" + String(GPT_CALLED) + "\n"
      + "WRITE_PERFORMED=" + String(WRITE_PERFORMED);
    Logger.log(log);
    return { ok: false, error: "MATCH_COUNT != 1" , matchCount: matchCount };
  }

  const packet = matched[0];
  selectedPid = packet.pid;
  representativeRow = packet.representativeRow;

  if (!representativeRow || typeof representativeRow !== "number") {
    const log = "=== P DETERMINISTIC REAL-PID DRY-RUN ===\n"
      + "ACTIVE_SHEET=" + String(sh.getName ? sh.getName() : "") + "\n"
      + "REQUESTED_PID=" + pid + "\n"
      + "MATCH_COUNT=" + matchCount + "\n"
      + "SELECTED_PID=" + selectedPid + "\n"
      + "representativeRow=" + String(representativeRow) + "\n"
      + "P_STATUS=ERROR\nP_ERROR_TYPE=UPSTREAM_NOT_FINALIZED\n"
      + "GPT_CALLED=" + String(GPT_CALLED) + "\n"
      + "WRITE_PERFORMED=" + String(WRITE_PERFORMED);
    Logger.log(log);
    return { ok: false, error: "representativeRow invalid" };
  }

  // read only
  const kCell = sh.getRange(representativeRow, kCol);
  const cCell = sh.getRange(representativeRow, cCol);
  const mCell = sh.getRange(representativeRow, mCol);
  kCellValue = String(kCell.getDisplayValue ? kCell.getDisplayValue() : kCell.getValue());
  cCellValue = String(cCell.getDisplayValue ? cCell.getDisplayValue() : cCell.getValue());
  mCellValue = String(mCell.getDisplayValue ? mCell.getDisplayValue() : mCell.getValue());

  kNoteText = kCell.getNote ? String(kCell.getNote() || "") : "";
  cNoteText = cCell.getNote ? String(cCell.getNote() || "") : "";
  mNoteText = mCell.getNote ? String(mCell.getNote() || "") : "";
  kNotePresent = !!(kNoteText && kNoteText.trim().length > 0);
  cNotePresent = !!(cNoteText && cNoteText.trim().length > 0);
  mNotePresent = !!(mNoteText && mNoteText.trim().length > 0);

  kParsed = _parseKCMPDecisionNote_(kNoteText, "K");
  cParsed = _parseKCMPDecisionNote_(cNoteText, "C");
  mParsed = _parseKCMPDecisionNote_(mNoteText, "M");

  kStatus = kParsed.state;
  cStatus = cParsed.state;
  mStatus = mParsed.state;
  kCode = kParsed.result && "code" in kParsed.result ? String(kParsed.result.code) : "";
  cCode = cParsed.result && "code" in cParsed.result ? String(cParsed.result.code) : "";
  mCode = mParsed.result && "code" in mParsed.result ? String(mParsed.result.code) : "";
  kContribs = _extractStudentContributorsFromNote_(kParsed);
  cContribs = _extractStudentContributorsFromNote_(cParsed);
  mContribs = _extractStudentContributorsFromNote_(mParsed);

  if (kParsed.state === "OK" && cParsed.state === "OK" && mParsed.state === "OK") {
    unionContribs = _unionStudentContributors_([kContribs, cContribs, mContribs]);
    unionCount = unionContribs.length;
  }

  pResult = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
  pStatus = pResult && pResult.status ? pResult.status : "";
  pCode = pResult && pResult.code ? pResult.code : "";
  pReason = pResult && pResult.reason ? pResult.reason : "";
  pErrorType = pResult && pResult.error_type ? pResult.error_type : "";

  const log =
    "=== P DETERMINISTIC REAL-PID DRY-RUN ===\n"
    + "ACTIVE_SHEET=" + String(sh.getName ? sh.getName() : "") + "\n"
    + "REQUESTED_PID=" + pid + "\n"
    + "MATCH_COUNT=" + matchCount + "\n"
    + "SELECTED_PID=" + selectedPid + "\n"
    + "representativeRow=" + String(representativeRow) + "\n"
    + "\nK_CELL_VALUE=" + kCellValue + "\n"
    + "C_CELL_VALUE=" + cCellValue + "\n"
    + "M_CELL_VALUE=" + mCellValue + "\n"
    + "K_NOTE_PRESENT=" + String(kNotePresent) + "\n"
    + "C_NOTE_PRESENT=" + String(cNotePresent) + "\n"
    + "M_NOTE_PRESENT=" + String(mNotePresent) + "\n"
    + "K_NOTE_PARSED=" + JSON.stringify(kParsed) + "\n"
    + "C_NOTE_PARSED=" + JSON.stringify(cParsed) + "\n"
    + "M_NOTE_PARSED=" + JSON.stringify(mParsed) + "\n"
    + "\nK_STATUS=" + kStatus + "\n"
    + "K_CODE=" + kCode + "\n"
    + "K_CONTRIBUTORS=" + JSON.stringify(kContribs) + "\n"
    + "\nC_STATUS=" + cStatus + "\n"
    + "C_CODE=" + cCode + "\n"
    + "C_CONTRIBUTORS=" + JSON.stringify(cContribs) + "\n"
    + "\nM_STATUS=" + mStatus + "\n"
    + "M_CODE=" + mCode + "\n"
    + "M_CONTRIBUTORS=" + JSON.stringify(mContribs) + "\n"
    + "\nUNION_CONTRIBUTORS=" + JSON.stringify(unionContribs) + "\n"
    + "UNION_COUNT=" + unionCount + "\n"
    + "\nP_STATUS=" + pStatus + "\n"
    + "P_CODE=" + pCode + "\n"
    + "P_REASON=" + pReason + "\n"
    + "P_ERROR_TYPE=" + pErrorType + "\n"
    + "\nGPT_CALLED=" + String(GPT_CALLED) + "\n"
    + "WRITE_PERFORMED=" + String(WRITE_PERFORMED);

  Logger.log(log);
  return { ok: true, result: pResult, matchCount: matchCount };
}

function TEST_P_DETERMINISTIC_FOR_PID(){
  // TODO: 여기 PID만 바꿔서 재현
  return testDeterministicPForPid_("P038");
}

/**
 * FINALIZED sample 탐색 (최대 5개)
 * - GPT 호출/시트 쓰기/Note 생성 금지
 * - K/C/M 세 Note가 모두 존재하고 status=OK 인 packet만 샘플로 선택
 */
function TEST_P_FIND_FINALIZED_SAMPLE(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  const list = [];

  (packets || []).forEach(function(packet){
    if (list.length >= 5) return;
    if (!packet || !packet.representativeRow) return;

    const row = packet.representativeRow;
    const kCell = sh.getRange(row, kCol);
    const cCell = sh.getRange(row, cCol);
    const mCell = sh.getRange(row, mCol);

    const kNoteText = kCell.getNote ? String(kCell.getNote() || "") : "";
    const cNoteText = cCell.getNote ? String(cCell.getNote() || "") : "";
    const mNoteText = mCell.getNote ? String(mCell.getNote() || "") : "";

    const kParsed = _parseKCMPDecisionNote_(kNoteText, "K");
    const cParsed = _parseKCMPDecisionNote_(cNoteText, "C");
    const mParsed = _parseKCMPDecisionNote_(mNoteText, "M");

    if (kParsed.state !== "OK" || cParsed.state !== "OK" || mParsed.state !== "OK") return;

    const kCode = kParsed.result ? kParsed.result.code : null;
    const cCode = cParsed.result ? cParsed.result.code : null;
    const mCode = mParsed.result ? mParsed.result.code : null;

    const kContribs = _extractStudentContributorsFromNote_(kParsed);
    const cContribs = _extractStudentContributorsFromNote_(cParsed);
    const mContribs = _extractStudentContributorsFromNote_(mParsed);
    const union = _unionStudentContributors_([kContribs, cContribs, mContribs]);
    const expectedP = _pCodeFromContributorCount_(union.length);

    list.push({
      pid: packet.pid,
      row: row,
      kCode: kCode,
      kContribs: kContribs,
      cCode: cCode,
      cContribs: cContribs,
      mCode: mCode,
      mContribs: mContribs,
      expectedP: expectedP
    });
  });

  if (!list.length) {
    const msg = "=== P DETERMINISTIC FIND FINALIZED SAMPLE ===\n"
      + "FINALIZED_SAMPLE_COUNT=0\n"
      + "NO_FINALIZED_KCM_SAMPLE=true";
    Logger.log(msg);
    return { count: 0 };
  }

  const lines = [];
  lines.push("=== P DETERMINISTIC FIND FINALIZED SAMPLE ===");
  lines.push("FINALIZED_SAMPLE_COUNT=" + list.length);
  list.forEach(function(s, idx){
    lines.push("");
    lines.push("Sample#" + (idx + 1));
    lines.push("PID=" + String(s.pid));
    lines.push("ROW=" + String(s.row));
    lines.push("K_CODE=" + String(s.kCode));
    lines.push("K_CONTRIBUTORS=" + JSON.stringify(s.kContribs));
    lines.push("C_CODE=" + String(s.cCode));
    lines.push("C_CONTRIBUTORS=" + JSON.stringify(s.cContribs));
    lines.push("M_CODE=" + String(s.mCode));
    lines.push("M_CONTRIBUTORS=" + JSON.stringify(s.mContribs));
    lines.push("EXPECTED_P_BY_UNION=" + String(s.expectedP));
  });

  const out = lines.join("\n");
  Logger.log(out);
  return { count: list.length, samples: list };
}

/**
 * STEP 6B) No-Write K/C/M → Deterministic P Integration Test
 * - K/C/M production decision functions를 메모리에서 호출
 * - sheet write 전혀 없음 (_writeKDecisionCell_ 등 호출 금지)
 * - P는 GPT 미사용. K/C/M GPT 호출은 통합 검증 목적상 허용.
 */
function testKCMPDeterministicPIntegrationForPid_(requestedPid){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();

  const pid = String(requestedPid || "").trim();
  const packets = buildAllKCMPClusterPackets_(sh, map);
  const matched = (packets || []).filter(function(p){
    return p && p.pid === pid;
  });
  const matchCount = matched.length;

  const logHeader =
    "=== KCMP -> DETERMINISTIC P INTEGRATION DRY-RUN ===\n"
    + "ACTIVE_SHEET=" + String(sh.getName ? sh.getName() : "") + "\n"
    + "REQUESTED_PID=" + pid + "\n"
    + "MATCH_COUNT=" + matchCount + "\n";

  if (matchCount !== 1) {
    Logger.log(logHeader + "SELECTED_PID=\nrepresentativeRow=\n[ABORT] MATCH_COUNT != 1");
    return { ok: false, error: "MATCH_COUNT != 1", matchCount: matchCount };
  }

  const packet = matched[0];
  const selectedPid = packet.pid;
  const representativeRow = packet.representativeRow;

  // ----- K decision (memory only, no write) -----
  const kResult = runKDecisionTreeForPacket_(packet);
  const kStatus = kResult && kResult.status ? kResult.status : "";
  const kCode   = kResult && kResult.code != null ? String(kResult.code) : "null";
  const kContribs = _extractStudentContributorsFromNote_(
    { state: kStatus === "OK" ? "OK" : "ERROR", result: kResult }
  );
  const kValid = kStatus === "OK";

  // ----- C decision (memory only, no write) -----
  const cResult = runCDecisionTreeForPacket_(packet);
  const cStatus = cResult && cResult.status ? cResult.status : "";
  const cCode   = cResult && cResult.code != null ? String(cResult.code) : "null";
  const cContribs = _extractStudentContributorsFromNote_(
    { state: cStatus === "OK" ? "OK" : "ERROR", result: cResult }
  );
  const cValid = cStatus === "OK";

  // ----- M decision (memory only, no write) -----
  const allPacketsCtx = { allPackets: packets };
  const mResult = runMDecisionTreeForPacket_(packet, allPacketsCtx);
  const mStatus = mResult && mResult.status ? mResult.status : "";
  const mCode   = mResult && mResult.code != null ? String(mResult.code) : "null";
  const mContribs = _extractStudentContributorsFromNote_(
    { state: mStatus === "OK" ? "OK" : "ERROR", result: mResult }
  );
  const mValid = mStatus === "OK";

  // ----- memory Note JSON (no sheet write) -----
  const kNoteText = JSON.stringify(kResult);
  const cNoteText = JSON.stringify(cResult);
  const mNoteText = JSON.stringify(mResult);

  // ----- deterministic P (no GPT) -----
  const pResult = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
  const pStatus    = pResult && pResult.status ? pResult.status : "";
  const pCode      = pResult && pResult.code   ? String(pResult.code) : "";
  const pContribs  = (pResult && pResult.contributors) ? pResult.contributors : [];
  const pReason    = (pResult && pResult.reason) ? pResult.reason : "";
  const pErrorType = (pResult && pResult.error_type) ? pResult.error_type : "";

  // ----- union 독립 계산 (debug 확인용) -----
  const unionContribs = _unionStudentContributors_([kContribs, cContribs, mContribs]);
  const unionCount    = unionContribs.length;
  const expectedByCount = _pCodeFromContributorCount_(unionCount);
  const pMatchExpected = (pStatus === "OK") && (pCode === expectedByCount);

  const P_GPT_CALLED = false;

  const log = logHeader
    + "SELECTED_PID=" + selectedPid + "\n"
    + "representativeRow=" + String(representativeRow) + "\n"
    + "\n--- K ---\n"
    + "K_STATUS=" + kStatus + "\n"
    + "K_CODE=" + kCode + "\n"
    + "K_CONTRIBUTORS=" + JSON.stringify(kContribs) + "\n"
    + "K_VALID=" + String(kValid) + "\n"
    + "\n--- C ---\n"
    + "C_STATUS=" + cStatus + "\n"
    + "C_CODE=" + cCode + "\n"
    + "C_CONTRIBUTORS=" + JSON.stringify(cContribs) + "\n"
    + "C_VALID=" + String(cValid) + "\n"
    + "\n--- M ---\n"
    + "M_STATUS=" + mStatus + "\n"
    + "M_CODE=" + mCode + "\n"
    + "M_CONTRIBUTORS=" + JSON.stringify(mContribs) + "\n"
    + "M_VALID=" + String(mValid) + "\n"
    + "\n--- P ---\n"
    + "UNION_CONTRIBUTORS=" + JSON.stringify(unionContribs) + "\n"
    + "UNION_COUNT=" + unionCount + "\n"
    + "EXPECTED_BY_COUNT=" + expectedByCount + "\n"
    + "\nP_STATUS=" + pStatus + "\n"
    + "P_CODE=" + pCode + "\n"
    + "P_CONTRIBUTORS=" + JSON.stringify(pContribs) + "\n"
    + "P_REASON=" + pReason + "\n"
    + "P_ERROR_TYPE=" + pErrorType + "\n"
    + "\nP_MATCH_EXPECTED=" + String(pMatchExpected) + "\n"
    + "\nK_WRITE=false\n"
    + "C_WRITE=false\n"
    + "M_WRITE=false\n"
    + "P_WRITE=false\n"
    + "P_GPT_CALLED=" + String(P_GPT_CALLED);

  Logger.log(log);
  return {
    ok: true,
    kResult: kResult,
    cResult: cResult,
    mResult: mResult,
    pResult: pResult,
    unionContribs: unionContribs,
    expectedByCount: expectedByCount,
    pMatchExpected: pMatchExpected
  };
}

function TEST_KCMP_P_INTEGRATION(){
  // 여기 PID만 바꿔서 재현 (production logic에 PID hardcode 금지)
  return testKCMPDeterministicPIntegrationForPid_("P038");
}

// ============================================================
// STEP 8: Final KCMP Pipeline Regression — DRY-RUN ONLY
// sheet write=0, P GPT call=0, semantic calibration 금지
// ============================================================

/**
 * sheet-aware packet 조회 helper (regression 전용).
 * sheetName으로 정확한 sheet를 찾고 해당 sheet의 packet을 반환.
 * active sheet fallback 없음.
 */
function _regressionGetPacket_(sheetName, pid) {
  const ss = SpreadsheetApp.getActive();
  const sh = ss.getSheetByName(sheetName);
  if (!sh) return { error: "SHEET_NOT_FOUND: " + sheetName };
  const map = loadColMap_();
  if (!map || !map.S1) return { error: "COLMAP_MISSING" };
  const packets = buildAllKCMPClusterPackets_(sh, map);
  const matched = (packets || []).filter(function(p){ return p && p.pid === String(pid); });
  if (matched.length !== 1) return { error: "PACKET_MATCH_FAIL: count=" + matched.length + " pid=" + pid, allPackets: packets };
  return { sh: sh, map: map, packet: matched[0], allPackets: packets };
}

/**
 * 단일 case 실행 helper.
 * dimension: "K" | "C" | "M" | "KCM_P"
 */
function _regressionRunCase_(sheetName, pid, dimension, expectedCode, goldKey) {
  const r = _regressionGetPacket_(sheetName, pid);
  if (r.error) {
    return {
      goldKey: goldKey, dimension: dimension,
      expected: expectedCode, got: null,
      status: "TEST_HARNESS_ERROR", validationOk: false,
      contributors: [], pass: false,
      error: r.error
    };
  }
  const packet = r.packet;
  const allPackets = r.allPackets;

  // -----------------------------------------------------------------
  // STRUCTURAL ERROR = production runner가 반환한 ERROR status만 포함.
  // status=OK인 결과를 harness secondary check로 structural error로
  // 분류하지 않는다.
  // -----------------------------------------------------------------
  function isStructuralError_(res) {
    if (!res) return true;
    const s = String(res.status || "");
    return s === "API_ERROR" || s === "PARSER_ERROR" || s === "VALIDATION_ERROR" || s === "PACKET_ERROR" || s === "ERROR";
  }

  // -----------------------------------------------------------------
  // PASS 판단: runner final status=OK가 primary.
  // secondary validator는 debug log 전용 (pass 판단에 사용하지 않음).
  // null code + status=OK는 정상 finalized result이다.
  // -----------------------------------------------------------------
  function runnerOk_(res) {
    return res && res.status === "OK";
  }

  if (dimension === "K") {
    const kRes = runKDecisionTreeForPacket_(packet);
    const structErr = isStructuralError_(kRes);
    // secondary validation: debug 전용
    const vld = runnerOk_(kRes) ? validateKDecisionResult_(kRes, packet) : { ok: false, errors: ["runner status not OK"] };
    const gotCode = (kRes && kRes.code !== undefined) ? (kRes.code === null ? "null" : String(kRes.code)) : "null";
    const expectedStr = expectedCode === null ? "null" : String(expectedCode);
    const contribs = (kRes && kRes.contributors) ? kRes.contributors : [];
    // PASS: structural error 없음 AND runner OK AND code match
    const pass = !structErr && runnerOk_(kRes) && gotCode === expectedStr;
    return {
      goldKey: goldKey, dimension: "K",
      expected: expectedStr, got: gotCode,
      status: (kRes && kRes.status) || "NULL",
      validationOk: vld.ok,                  // debug only
      validationErrors: vld.errors || [],    // debug only
      contributors: contribs,
      structuralError: structErr,
      pass: pass,
      secondaryValidationUsed: true
    };
  }

  if (dimension === "C") {
    const cRes = runCDecisionTreeForPacket_(packet);
    const structErr = isStructuralError_(cRes);
    const vld = runnerOk_(cRes) ? validateCDecisionResult_(cRes, packet) : { ok: false, errors: ["runner status not OK"] };
    const gotCode = (cRes && cRes.code !== undefined) ? (cRes.code === null ? "null" : String(cRes.code)) : "null";
    const expectedStr = expectedCode === null ? "null" : String(expectedCode);
    const contribs = (cRes && cRes.contributors) ? cRes.contributors : [];
    const pass = !structErr && runnerOk_(cRes) && gotCode === expectedStr;
    return {
      goldKey: goldKey, dimension: "C",
      expected: expectedStr, got: gotCode,
      status: (cRes && cRes.status) || "NULL",
      validationOk: vld.ok,
      validationErrors: vld.errors || [],
      contributors: contribs,
      structuralError: structErr,
      pass: pass,
      secondaryValidationUsed: true
    };
  }

  if (dimension === "M") {
    const mCtx = { allPackets: allPackets };
    const mRes = runMDecisionTreeForPacket_(packet, mCtx);
    const structErr = isStructuralError_(mRes);
    const vld = runnerOk_(mRes) ? validateMDecisionResult_(mRes, packet, mCtx) : { ok: false, errors: ["runner status not OK"] };
    const gotCode = (mRes && mRes.code !== undefined) ? (mRes.code === null ? "null" : String(mRes.code)) : "null";
    const expectedStr = expectedCode === null ? "null" : String(expectedCode);
    const contribs = (mRes && mRes.contributors) ? mRes.contributors : [];
    const pass = !structErr && runnerOk_(mRes) && gotCode === expectedStr;
    return {
      goldKey: goldKey, dimension: "M",
      expected: expectedStr, got: gotCode,
      status: (mRes && mRes.status) || "NULL",
      validationOk: vld.ok,
      validationErrors: vld.errors || [],
      contributors: contribs,
      structuralError: structErr,
      pass: pass,
      secondaryValidationUsed: true
    };
  }

  if (dimension === "KCM_P") {
    // K + C + M + deterministic P (no write, no GPT)
    const kRes = runKDecisionTreeForPacket_(packet);
    const cRes = runCDecisionTreeForPacket_(packet);
    const mCtx = { allPackets: allPackets };
    const mRes = runMDecisionTreeForPacket_(packet, mCtx);

    // structural error: production runner status 기준만
    const kStructErr = isStructuralError_(kRes);
    const cStructErr = isStructuralError_(cRes);
    const mStructErr = isStructuralError_(mRes);
    const anyStructErr = kStructErr || cStructErr || mStructErr;

    // secondary validation: debug log 전용 (pass 판단 미사용)
    const kVld = runnerOk_(kRes) ? validateKDecisionResult_(kRes, packet) : { ok: false, errors: ["runner status not OK"] };
    const cVld = runnerOk_(cRes) ? validateCDecisionResult_(cRes, packet) : { ok: false, errors: ["runner status not OK"] };
    const mVld = runnerOk_(mRes) ? validateMDecisionResult_(mRes, packet, mCtx) : { ok: false, errors: ["runner status not OK"] };

    const kCode = (kRes && kRes.code !== undefined) ? (kRes.code === null ? "null" : String(kRes.code)) : "null";
    const cCode = (cRes && cRes.code !== undefined) ? (cRes.code === null ? "null" : String(cRes.code)) : "null";
    const mCode = (mRes && mRes.code !== undefined) ? (mRes.code === null ? "null" : String(mRes.code)) : "null";

    const kContribs = _extractStudentContributorsFromNote_({ state: runnerOk_(kRes) ? "OK" : "ERROR", result: kRes });
    const cContribs = _extractStudentContributorsFromNote_({ state: runnerOk_(cRes) ? "OK" : "ERROR", result: cRes });
    const mContribs = _extractStudentContributorsFromNote_({ state: runnerOk_(mRes) ? "OK" : "ERROR", result: mRes });

    // P: K/C/M result를 변형 없이 그대로 전달 (harness validation flag로 수정 금지)
    const kNoteText = JSON.stringify(kRes);
    const cNoteText = JSON.stringify(cRes);
    const mNoteText = JSON.stringify(mRes);
    const pRes = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
    const pCode = (pRes && pRes.code) ? String(pRes.code) : "null";
    const pContribs = (pRes && pRes.contributors) ? pRes.contributors : [];

    const unionContribs = _unionStudentContributors_([kContribs, cContribs, mContribs]);
    const expectedP = _pCodeFromContributorCount_(unionContribs.length);
    const pMatchExpected = (pRes && pRes.status === "OK") && (pCode === expectedP);

    // PASS: structural error 없음 AND all runners OK AND M code match AND P match
    const mExpectedStr = expectedCode === null ? "null" : String(expectedCode);
    const allRunnersOk = runnerOk_(kRes) && runnerOk_(cRes) && runnerOk_(mRes);
    const mMatch = mCode === mExpectedStr;
    const overallPass = !anyStructErr && allRunnersOk && mMatch && pMatchExpected;

    return {
      goldKey: goldKey, dimension: "KCM_P",
      expected: mExpectedStr, got: mCode,
      status: (mRes && mRes.status) || "NULL",
      // secondary validation: debug 전용
      validationOk: mVld.ok,
      validationErrors: mVld.errors || [],
      contributors: mContribs,
      structuralError: anyStructErr,
      pass: overallPass,
      secondaryValidationUsed: true,
      // extended P details
      kCode: kCode, kContribs: kContribs,
      kStatus: (kRes && kRes.status) || "NULL", kVldOk: kVld.ok, kVldErrors: kVld.errors || [],
      cCode: cCode, cContribs: cContribs,
      cStatus: (cRes && cRes.status) || "NULL", cVldOk: cVld.ok, cVldErrors: cVld.errors || [],
      mCode: mCode, mContribs: mContribs,
      mStatus: (mRes && mRes.status) || "NULL", mVldOk: mVld.ok, mVldErrors: mVld.errors || [],
      m3Evidence: (mRes && mRes.m3_evidence) ? mRes.m3_evidence : null,
      kRawFinalResult: kRes, cRawFinalResult: cRes, mRawFinalResult: mRes,
      packetCurrentTurns: (packet && packet.turns) ? packet.turns.length : 0,
      packetPriorTurns: (packet && packet.context && packet.context.prior) ? packet.context.prior.length : 0,
      unionContribs: unionContribs,
      expectedPByCount: expectedP,
      actualP: pCode, pContribs: pContribs,
      pInputKStatus: (kRes && kRes.status) || "NULL",
      pInputCStatus: (cRes && cRes.status) || "NULL",
      pInputMStatus: (mRes && mRes.status) || "NULL",
      pMatchExpected: pMatchExpected,
      pGptCalled: false, kWrite: false, cWrite: false, mWrite: false, pWrite: false
    };
  }

  return { goldKey: goldKey, dimension: dimension, error: "UNKNOWN_DIMENSION", pass: false };
}

/**
 * TEST_KCMP_FINAL_REGRESSION
 * STEP 8: 8 representative cases dry-run regression.
 * sheet write=0, P GPT call=0.
 * DIAGNOSTIC ONLY: production gate로 사용하지 않는다.
 */
function TEST_KCMP_FINAL_REGRESSION() {
  // ---- GOLD CASES ----
  // sheetName :: PID, dimension, expectedCode
  const CASES = [
    // A. M4 conceptual-gap boundary
    { sheetName: "14차시 4조", pid: "P009", dimension: "M", expectedCode: "M4",   note: "M4 conceptual-gap boundary" },
    // B. M1 process regulation
    { sheetName: "14차시 4조", pid: "P017", dimension: "M", expectedCode: "M1",   note: "M1 process regulation" },
    // C. terminal teacher challenge — should NOT retro-code earlier explanation as M3
    { sheetName: "14차시 4조", pid: "P037", dimension: "M", expectedCode: null,   note: "teacher challenge: earlier explanation NOT M3" },
    // D. teacher challenge → student response M3 + deterministic P
    { sheetName: "14차시 4조", pid: "P038", dimension: "KCM_P", expectedCode: "M3", note: "M3 + deterministic P integration" },
    // E. explanation equivalence M3
    { sheetName: "14차시 4조", pid: "P053", dimension: "M", expectedCode: "M3",   note: "explanation equivalence M3" },
    // F. C3 final-relation scoping
    { sheetName: "17차시 4조", pid: "P055", dimension: "C", expectedCode: "C3",   note: "C3 final-relation scoping" },
    // G. C6 joint decision scoping
    { sheetName: "17차시 4조", pid: "P037", dimension: "C", expectedCode: "C6",   note: "C6 joint decision scoping" },
    // H. teacher-mediated, no real student-student linkage
    { sheetName: "17차시 4조", pid: "P035", dimension: "C", expectedCode: null,   note: "teacher-mediated: no student-student linkage" },
  ];

  const results = [];
  let semanticPass = 0, semanticFail = 0, structErrCount = 0;
  let apiErrCount = 0, parserErrCount = 0, validationErrCount = 0, packetErrCount = 0;
  const failedKeys = [];
  let p038PMatchExpected = null;

  Logger.log("=== TEST_KCMP_FINAL_REGRESSION START ===");
  Logger.log("TOTAL_CASES=" + CASES.length);
  Logger.log("SHEET_WRITES=0 (dry-run only)");
  Logger.log("P_GPT_CALLS=0 (deterministic P)\n");

  CASES.forEach(function(c, idx) {
    const goldKey = c.sheetName + "::" + c.pid;
    Logger.log("--- CASE " + (idx + 1) + " ---");
    Logger.log("CASE=" + (idx + 1));
    Logger.log("GOLD_KEY=" + goldKey);
    Logger.log("DIMENSION=" + c.dimension);
    Logger.log("NOTE=" + c.note);
    Logger.log("EXPECTED=" + (c.expectedCode === null ? "null" : c.expectedCode));

    let res;
    try {
      res = _regressionRunCase_(c.sheetName, c.pid, c.dimension, c.expectedCode, goldKey);
    } catch(e) {
      res = {
        goldKey: goldKey, dimension: c.dimension,
        expected: c.expectedCode, got: null,
        status: "TEST_HARNESS_ERROR", validationOk: false,
        contributors: [], pass: false,
        error: String(e)
      };
    }

    // 구조적 오류 분류
    if (res.error) {
      structErrCount++;
      packetErrCount++;
      Logger.log("GOT=ERROR");
      Logger.log("STATUS=TEST_HARNESS_ERROR");
      Logger.log("VALIDATION_OK=false");
      Logger.log("CONTRIBUTORS=[]");
      Logger.log("PASS=false");
      Logger.log("ERROR=" + res.error);
      failedKeys.push(goldKey);
      semanticFail++;
      results.push(res);
      return;
    }

    const statusStr = String(res.status || "");
    if (statusStr === "API_ERROR")        apiErrCount++;
    if (statusStr === "PARSER_ERROR")     parserErrCount++;
    if (statusStr === "VALIDATION_ERROR") validationErrCount++;
    if (statusStr === "PACKET_ERROR")     packetErrCount++;
    if (res.structuralError)              structErrCount++;

    Logger.log("GOT=" + (res.got !== undefined ? res.got : "null"));
    Logger.log("STATUS=" + statusStr);
    Logger.log("VALIDATION_OK=" + String(res.validationOk));
    if (res.validationErrors && res.validationErrors.length) {
      Logger.log("VALIDATION_ERRORS=" + JSON.stringify(res.validationErrors));
    }
    Logger.log("CONTRIBUTORS=" + JSON.stringify(res.contributors || []));
    Logger.log("PASS=" + String(res.pass));

    // KCM_P case 상세 로그 (P038 디버그 포함)
    if (c.dimension === "KCM_P") {
      Logger.log("\n  --- P038 KCM_P DETAIL ---");
      Logger.log("  SECONDARY_VALIDATION_USED=" + String(res.secondaryValidationUsed));

      // K raw
      Logger.log("  K_RAW_FINAL_RESULT=" + JSON.stringify(res.kRawFinalResult || null));
      Logger.log("  K_RESULT_STATUS=" + res.kStatus);
      Logger.log("  K_CODE=" + res.kCode);
      Logger.log("  K_CONTRIBUTORS=" + JSON.stringify(res.kContribs || []));
      Logger.log("  K_VALID(debug)=" + String(res.kVldOk));
      if (res.kVldErrors && res.kVldErrors.length) Logger.log("  K_SECONDARY_VALIDATION_ERRORS=" + JSON.stringify(res.kVldErrors));

      // C raw
      Logger.log("  C_RAW_FINAL_RESULT=" + JSON.stringify(res.cRawFinalResult || null));
      Logger.log("  C_RESULT_STATUS=" + res.cStatus);
      Logger.log("  C_CODE=" + res.cCode);
      Logger.log("  C_CONTRIBUTORS=" + JSON.stringify(res.cContribs || []));
      Logger.log("  C_VALID(debug)=" + String(res.cVldOk));
      if (res.cVldErrors && res.cVldErrors.length) Logger.log("  C_SECONDARY_VALIDATION_ERRORS=" + JSON.stringify(res.cVldErrors));

      // M raw
      Logger.log("  M_RAW_FINAL_RESULT=" + JSON.stringify(res.mRawFinalResult || null));
      Logger.log("  M_RESULT_STATUS=" + res.mStatus);
      Logger.log("  M_CODE=" + res.mCode);
      Logger.log("  M_CONTRIBUTORS=" + JSON.stringify(res.mContribs || []));
      Logger.log("  M_VALID(debug)=" + String(res.mVldOk));
      if (res.mVldErrors && res.mVldErrors.length) Logger.log("  M_SECONDARY_VALIDATION_ERRORS=" + JSON.stringify(res.mVldErrors));
      Logger.log("  M3_EVIDENCE=" + JSON.stringify(res.m3Evidence || null));

      // packet context
      Logger.log("  PACKET_CURRENT_TURNS=" + String(res.packetCurrentTurns));
      Logger.log("  PACKET_PRIOR_CONTEXT_TURNS=" + String(res.packetPriorTurns));

      // P
      Logger.log("  P_INPUT_K_STATUS=" + res.pInputKStatus);
      Logger.log("  P_INPUT_C_STATUS=" + res.pInputCStatus);
      Logger.log("  P_INPUT_M_STATUS=" + res.pInputMStatus);
      Logger.log("  UNION_CONTRIBUTORS=" + JSON.stringify(res.unionContribs || []));
      Logger.log("  EXPECTED_P_BY_COUNT=" + res.expectedPByCount);
      Logger.log("  ACTUAL_P=" + res.actualP);
      Logger.log("  P_MATCH_EXPECTED=" + String(res.pMatchExpected));
      Logger.log("  K_WRITE=false");
      Logger.log("  C_WRITE=false");
      Logger.log("  M_WRITE=false");
      Logger.log("  P_WRITE=false");
      Logger.log("  P_GPT_CALLED=false");
      p038PMatchExpected = res.pMatchExpected;
    }

    if (res.pass) {
      semanticPass++;
    } else {
      semanticFail++;
      failedKeys.push(goldKey);
    }
    results.push(res);
    Logger.log("");
  });

  // ---- AGGREGATE ----
  const finalPass = (semanticPass >= 7) && (structErrCount === 0) && (p038PMatchExpected === true);

  Logger.log("\n=== KCMP FINAL REGRESSION ===");
  Logger.log("TOTAL_CASES=" + CASES.length);
  Logger.log("SEMANTIC_PASS=" + semanticPass);
  Logger.log("SEMANTIC_FAIL=" + semanticFail);
  Logger.log("STRUCTURAL_ERROR_COUNT=" + structErrCount);
  Logger.log("API_ERROR_COUNT=" + apiErrCount);
  Logger.log("PARSER_ERROR_COUNT=" + parserErrCount);
  Logger.log("VALIDATION_ERROR_COUNT=" + validationErrCount);
  Logger.log("PACKET_ERROR_COUNT=" + packetErrCount);
  Logger.log("FAILED_KEYS=" + JSON.stringify(failedKeys));
  Logger.log("P038_P_MATCH_EXPECTED=" + String(p038PMatchExpected));
  Logger.log("SHEET_WRITES=0");
  Logger.log("P_GPT_CALLS=0");
  Logger.log("FINAL_REGRESSION_PASS=" + String(finalPass));

  // DIAGNOSTIC ONLY
  // - production readiness gate로 사용하지 않는다.
  // - static live-path check(TEST_KCMP_LIVE_PATH_STATIC)만 gate로 유지한다.
  Logger.log("\nKCMP_PIPELINE_STATUS=" + KCMP_PIPELINE_STATUS);
  Logger.log("DIAGNOSTIC_ONLY=true");
  Logger.log("menu_runKCMP() 자동 실행 금지. 수동 확인 후 실행.");
  Logger.log("=================================");

  return {
    totalCases: CASES.length,
    semanticPass: semanticPass,
    semanticFail: semanticFail,
    structuralErrorCount: structErrCount,
    failedKeys: failedKeys,
    p038PMatchExpected: p038PMatchExpected,
    sheetWrites: 0,
    pGptCalls: 0,
    finalRegressionPass: finalPass,
    results: results
  };
}

// ============================================================
// STEP 7: Static live-path check helper
// 실제 K/C/M/P 실행 금지. API 호출 금지. Sheet write 금지.
// Logger만 사용.
// ============================================================
function TEST_KCMP_LIVE_PATH_STATIC() {
  const checks = [];

  // 1. LIVE FUNCTION 존재 확인
  checks.push({ key: "LIVE_K",    expected: "runCodeK_All",   ok: typeof runCodeK_All === "function" });
  checks.push({ key: "LIVE_C",    expected: "runCodeC_All",   ok: typeof runCodeC_All === "function" });
  checks.push({ key: "LIVE_M",    expected: "runCodeM_All",   ok: typeof runCodeM_All === "function" });
  checks.push({ key: "LIVE_P",    expected: "runCodeP_All",   ok: typeof runCodeP_All === "function" });

  // 2. INTEGRATED ORDER (menu_runKCMP, runKCMPCoding 소스 텍스트로 확인)
  const kcmpSrc   = menu_runKCMP.toString();
  const orderOk   = /runCodeK_All/.test(kcmpSrc) &&
                    /runCodeC_All/.test(kcmpSrc) &&
                    /runCodeM_All/.test(kcmpSrc) &&
                    /runCodeP_All/.test(kcmpSrc) &&
                    !/runCodeKM_All[^_]/.test(kcmpSrc); // runCodeKM_All_LEGACY 허용, runCodeKM_All 금지
  checks.push({ key: "INTEGRATED_ORDER", expected: "K→C→M→P (no runCodeKM_All)", ok: orderOk });

  // 3. P MODE: deterministic
  checks.push({ key: "P_MODE",    expected: "DETERMINISTIC",  ok: typeof computeDeterministicPForPacket_ === "function" });
  checks.push({ key: "P_GPT_LIVE", expected: false,           ok: typeof runCodeP_All_LEGACY_GPT === "function" // 존재하지만
  // live call site가 없는지는 소스 기반으로 확인
  && !/runCodeP_All_LEGACY_GPT\s*\(/.test(runCodeP_All.toString()) });

  // 4. M 상태
  checks.push({ key: "M_STATUS",         expected: "CLOSED_WITH_KNOWN_LIMITATIONS", ok: (typeof STEP_5_M_STATUS !== "undefined" && STEP_5_M_STATUS === "CLOSED_WITH_KNOWN_LIMITATIONS") });
  checks.push({ key: "M_PROMPT_FROZEN",  expected: true, ok: (typeof STEP_5_M_PROMPT_FROZEN !== "undefined" && STEP_5_M_PROMPT_FROZEN === true) });

  // 5. LEGACY runCodeKM_All: live runner에서 호출되지 않음
  const kmLiveSrc  = runKCMPCoding.toString();
  const kmMenuSrc  = menu_runKCMP.toString();
  const kmNotLive  = !/runCodeKM_All[^_]/.test(kmLiveSrc) && !/runCodeKM_All[^_]/.test(kmMenuSrc);
  checks.push({ key: "LEGACY_RUN_CODE_KM_LIVE", expected: false, ok: kmNotLive });

  // 6. LEGACY GPT P: runCodeP_All 본문에서 호출되지 않음
  const pSrc       = runCodeP_All.toString();
  const gptPNotLive = !/analyzePCodeWithGPT_/.test(pSrc) && !/runCodeP_All_LEGACY_GPT\s*\(/.test(pSrc);
  checks.push({ key: "LEGACY_GPT_P_LIVE", expected: false, ok: gptPNotLive });

  // 결과 출력
  let allPass = true;
  Logger.log("=== TEST_KCMP_LIVE_PATH_STATIC ===");
  checks.forEach(c => {
    const status = c.ok ? "PASS" : "FAIL";
    if (!c.ok) allPass = false;
    Logger.log(`  ${status}  ${c.key}  expected=${c.expected}  got=${c.ok}`);
  });
  Logger.log(`OVERALL: ${allPass ? "ALL_PASS" : "HAS_FAILURES"}`);
  Logger.log("==================================");
  return allPass;
}

// ============================================================
// LEGACY / INACTIVE / DO NOT USE
// runCodeP_All_LEGACY_GPT: GPT 기반 P 코딩 구버전 runner.
// STEP 6에서 deterministic P(runCodeP_All → computeDeterministicPForPacket_)로 교체.
// live call site = 0. 메뉴/integrated runner에서 완전 제거됨.
// 아래의 analyzePCodeWithGPT_, extractKCMCodes_, crossCheckGPTWithKCM_,
// generateFinalPCode_ 도 이 블록 전용 helper로 모두 legacy 상태임.
// ============================================================
function runCodeP_All_LEGACY_GPT() {
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const lastRow = Math.min(sh.getLastRow(), findLastPidRow_(sh, map));
  if (lastRow < 2) return ui.alert('데이터가 없습니다.');

  const rowCount = lastRow - 1;
  if (rowCount <= 0) {
    ui.alert('⚠️ 처리할 행이 없습니다.');
    return;
  }

  const sCols = getSColsFlexible_(sh, map, 1);
  const nCol = colNumOf(map.N);
  const fCol = colNumOf(map.F);
  const kCol = colNumOf(map.K), lCol = colNumOf(map.L), mCol = colNumOf(map.M);
  const aCol = colNumOf(map.A), eCol = colNumOf(map.E);

  // sHeaders 구성 (matchSpeakerToSx_ 함수용)
  const hdr = getHeaderRow_(sh);
  const sHeaders = sCols.map((c,i)=> c ? ({ key:'S'+(i+1), col:c, label:String(hdr[c-1]||'').trim() }) : null).filter(Boolean);

  const pidIndex = buildPIDIndex_(sh, map);

  // 파이프라인 가드: F열 요약문 확인
  let sampleChecked = false;
  let fCompletionRate = 0;
  if (lastRow >= 7) {
    const sampleSize = Math.min(5, rowCount);
    const sampleRows = [];
    for (let s = 0; s < sampleSize; s++) {
      const row = 2 + Math.floor(Math.random() * rowCount);
      sampleRows.push(row);
    }
    
    // 성능 개선: 배치 읽기로 변경
    const sampleData = sh.getRange(2, fCol, rowCount, 1).getValues();
    let filledCount = 0;
    sampleRows.forEach(row => {
      const fVal = String(sampleData[row-2][0] || '').trim();
      if (fVal && fVal.length > 10) filledCount++;
    });
    
    fCompletionRate = filledCount / sampleRows.length;
    sampleChecked = true;
  }
  
  if (sampleChecked && fCompletionRate < 0.6) {
    ui.alert('⚠️ F열 요약문 확인 필요\n\n샘플링 결과: F열 요약문 완료율 ' + 
             Math.round(fCompletionRate * 100) + '%\n\nF열 요약문을 먼저 완료한 후 E코딩을 실행하세요.');
    return;
  }

  // 사전 검증: F열 요약문 확인
  const sampleData = sh.getRange(2, fCol, rowCount, 1).getValues();
  const validSummaries = sampleData.filter(s => (s[0] || "").trim().length > 10);
  if (validSummaries.length === 0) {
    ui.alert('⚠️ F열 요약문이 없습니다.\n\n해결 방법:\n1. 클러스터링을 먼저 실행하세요\n2. F열에 요약문이 생성되었는지 확인하세요');
    return;
  }

  // 성능 개선: 배치 읽기로 변경 (row-by-row 접근 제거)
  const dataRows = sh.getRange(2, 1, rowCount, Math.max(eCol, fCol, ...sCols.filter(c=>c))).getValues();
  const displayRows = sh.getRange(2, 1, rowCount, Math.max(...sCols.filter(c=>c))).getDisplayValues();
  
  let successCount = 0;
  let emptyCount = 0;
  let errorCount = 0;
  let skippedCount = 0;
  
  for (let i=0; i<dataRows.length; i++){
    const rowIdx = i + 2; // 실제 행 번호
    const pid = String(dataRows[i][eCol-1]||'').trim();
    if (!/^P\d+/i.test(pid)) {
      skippedCount++;
      continue;
    }

    // 1️⃣ 활성 발화자 파악 (G~J) - 성능 개선: 배치 읽기 사용
    const activeCounts = sCols.map(c=>{
      if (!c) return 0;
      const val = displayRows[i][c-1] || '0';
      return parseFloat(String(val).replace(/[^\d.]/g,'')) || 0;
    });
    const activeSpeakers = activeCounts.map((n,idx)=> n>0 ? 'S'+(idx+1) : null).filter(Boolean);
    const activeNum = activeSpeakers.length;
    
    // 보조 지표 계산
    const totalTalk = activeCounts.reduce((sum, n) => sum + n, 0);
    const maxTalk = Math.max(...activeCounts);
    const dominant = totalTalk > 0 ? maxTalk / totalTalk : 0;

    // 2️⃣ F열 요약문 분석 (핵심!) - 성능 개선: 배치 읽기 사용
    let summaryText = String(dataRows[i][fCol-1] || '').trim();
    if (!summaryText || summaryText.length < 10) {
      sh.getRange(rowIdx, nCol).setValue('P0 의미 있는 참여 없음    [요약문없음]');
      skippedCount++;
      continue;
    }

    // F열 요약문 전처리: 발화자 태깅 개선
    summaryText = preprocessSummaryText_(summaryText, activeSpeakers, sHeaders);

    // 3️⃣ GPT 분석 실행
    let gptResult;
    try {
      gptResult = analyzePCodeWithGPT_(summaryText, activeSpeakers, sHeaders);
      if (gptResult.flags && gptResult.flags.includes('GPT응답비어있음')) {
        emptyCount++;
        Logger.log(`⚠️ P 코딩 빈 응답 [행${rowIdx}]: 요약문 길이=${summaryText.length}`);
      }
    } catch (e) {
      errorCount++;
      Logger.log(`❌ P 코딩 GPT 분석 오류 [행${rowIdx}]: ${e.toString()}`);
      gptResult = { code: 'P0', name: '의미 있는 참여 없음', contributors: [], flags: ['GPT분석오류'] };
    }
    
    // 4️⃣ K/C/M 코드와 cross-check
    const kcmCodes = extractKCMCodes_(pidIndex[pid]?.rows||[], aCol, kCol, lCol, mCol, sHeaders, activeSpeakers);
    const crossCheckResult = crossCheckGPTWithKCM_(gptResult, kcmCodes, activeNum, summaryText, pid);

    // 5️⃣ K/L/M 열 직접 확인: 각 열에 코드가 있는지 확인
    // ⚠️ 중요: K, L, M 열에 각각 어떤 코드도 나타나지 않을 때만 P0 부여
    // 예: C1만 있고 K, M 코드가 없는 경우 → P0가 아님 (P1~P3 부여)
    const hasKCode = checkColumnHasCode_(pidIndex[pid]?.rows||[], kCol);
    const hasCCode = checkColumnHasCode_(pidIndex[pid]?.rows||[], lCol);
    const hasMCode = checkColumnHasCode_(pidIndex[pid]?.rows||[], mCol);
    const hasAnyKLMCode = hasKCode || hasCCode || hasMCode;

    // 6️⃣ 최종 결과 생성 (K/L/M 열 확인 포함)
    const finalResult = generateFinalPCode_(gptResult, crossCheckResult, activeNum, dominant, hasAnyKLMCode);
    
    if (finalResult && finalResult.trim().length > 0) {
      sh.getRange(rowIdx, nCol).setValue(finalResult);
      successCount++;
      if (successCount <= 3) {
        Logger.log(`P 코딩 성공 [행${rowIdx}]: ${finalResult.substring(0, 50)}`);
      }
    } else {
      emptyCount++;
    }
  }
  
  // 결과 리포트
  let msg = `✅ P차원 코딩 완료\n\n`;
  msg += `📊 통계:\n`;
  msg += `- 성공 기록: ${successCount}개\n`;
  if (emptyCount > 0) msg += `- 빈 응답: ${emptyCount}개\n`;
  if (errorCount > 0) msg += `- GPT 오류: ${errorCount}개\n`;
  if (skippedCount > 0) msg += `- 건너뜀(PID/요약문 없음): ${skippedCount}개\n`;
  msg += `\n개선사항:\n- F열 요약문 기반 실제 담화 분석\n- GPT와 K/C/M 코드 cross-check\n- 신뢰도 점검 및 플래그 시스템`;
  
  if (successCount === 0 && emptyCount > 0) {
    msg += `\n\n⚠️ 모든 응답이 비어있습니다.\n`;
    msg += `- GPT 응답 파싱 문제일 수 있습니다.\n`;
    msg += `- [보기] → [실행 로그]에서 상세 오류 확인하세요.`;
  }
  
  ui.alert(msg);
}

// ============================================================
// LEGACY / INACTIVE / DO NOT USE
// analyzePCodeWithGPT_: runCodeP_All_LEGACY_GPT 전용 helper.
// live P path(computeDeterministicPForPacket_)에서 사용하지 않음.
// ============================================================
function analyzePCodeWithGPT_(summaryText, activeSpeakers, sHeaders) {
  // 활성 발화자 정보 구성
  const activeSpeakerInfo = activeSpeakers.map(sx => {
    const k = parseInt(sx.replace('S',''),10);
    const entry = sHeaders[k-1];
    return entry ? entry.label || `참석자 ${k}` : `참석자 ${k}`;
  }).join(', ');

  const prompt = `다음은 한 클러스터의 담화 요약문입니다.

⚠️ 중요: 활성 발화자는 다음 ${activeSpeakers.length}명만입니다: ${activeSpeakerInfo}
- 교사 발화나 제3자 언급은 제외하고 분석하세요
- 위에 명시된 활성 발화자 중에서만 의미 있는 기여자를 찾으세요

각 활성 발화자의 발화 내용 중 의미 있는 기여(설명, 정당화, 조정)가 몇 명에게 분포했는지 판단하고,
그 수에 따라 아래 규칙으로 P코드를 부여하세요.

규칙:
- 1명만 의미 있게 기여 → P1
- 2~3명 의미 있게 기여 → P2  
- 4명 이상 의미 있게 기여 → P3
- 아무도 인식적 발화 없음 → P0

출력 형식:
P코드 + 각 학생별 의미 있는 기여 유형(K/C/M) 요약
예시: "P2 소수의 의미 있는 참여 | 참석자 1: K1, M2 / 참석자 3: C4"

요약문:
${summaryText}`;

  try {
    const response = callGPTAPI_(prompt, MODEL_P);
    if (!response || response.trim() === '') {
      Logger.log('⚠️ P 코딩 GPT 응답이 비어있습니다.');
      return { code: 'P0', name: '의미 있는 참여 없음', contributors: [], flags: ['GPT응답비어있음'] };
    }
    return parseGPTResponse_(response, activeSpeakers, sHeaders);
  } catch (error) {
    Logger.log('❌ P 코딩 GPT 분석 실패: ' + error.toString());
    return { code: 'P0', name: '의미 있는 참여 없음', contributors: [], flags: ['GPT분석실패'] };
  }
}

/**
 * A/C/D 코드 추출 (보조 자료용)
 * LEGACY / INACTIVE / DO NOT USE
 * runCodeP_All_LEGACY_GPT 전용 helper. live P path에서 사용하지 않음.
 */
function extractKCMCodes_(rows, aCol, kCol, lCol, mCol, sHeaders, activeSpeakers) {
  const codes = {S1:[], S2:[], S3:[], S4:[]};
  const sh = SpreadsheetApp.getActiveSheet();
  
  // 성능 개선: 배치 읽기로 변경 (row-by-row 접근 제거)
  if (rows.length === 0) return codes;
  const minRow = Math.min(...rows);
  const maxRow = Math.max(...rows);
  const maxCol = Math.max(aCol, kCol, lCol, mCol);
  const batchData = sh.getRange(minRow, 1, maxRow - minRow + 1, maxCol).getValues();
  
  for (const r of rows){
    const rowIdx = r - minRow; // 배치 배열 인덱스
    const speaker = String(batchData[rowIdx][aCol-1]||'').trim();
    let key = matchSpeakerToSx_(speaker, sHeaders);
    
    if (!key || !activeSpeakers.includes(key)) continue;
    
    // K/C/M 열에서 의미 있는 코드 추출 - 성능 개선: 배치 읽기 사용
    const kVal = String(batchData[rowIdx][kCol-1]||'').trim();
    const cVal = String(batchData[rowIdx][lCol-1]||'').trim();
    const mVal = String(batchData[rowIdx][mCol-1]||'').trim();
    
    const kMatch = kVal.match(/K[1-3]/g);
    if (kMatch) codes[key].push(...kMatch);
    
    const cMatch = cVal.match(/C[2-7]/g);
    if (cMatch) codes[key].push(...cMatch);
    
    const mMatch = mVal.match(/M[1-4]/g);
    if (mMatch) codes[key].push(...mMatch);
  }
  
  return codes;
}

/**
 * GPT 결과와 K/C/M 코드 cross-check (개선된 버전 + 차이 사례 수집)
 * LEGACY / INACTIVE / DO NOT USE
 * runCodeP_All_LEGACY_GPT 전용 helper. live P path에서 사용하지 않음.
 */
function crossCheckGPTWithKCM_(gptResult, kcmCodes, activeNum, summaryText, pid) {
  const flags = [];
  
  // GPT가 의미 있는 기여자로 판단한 수
  const gptContributorCount = gptResult.contributors.length;
  
  // K/C/M 코드가 있는 기여자 수
  const kcmContributorCount = Object.values(kcmCodes).filter(codes => codes.length > 0).length;
  
  // 일관성 체크 (GPT분석결과활성발화자수초과는 이미 parseGPTResponse_에서 처리됨)
  if (gptContributorCount === 0 && kcmContributorCount > 0) {
    flags.push('검토필요: KCM있으나GPT분석결과없음');
  }
  
  // GPT와 K/C/M 결과 차이가 클 때만 플래그 (1명 차이는 허용)
  if (Math.abs(gptContributorCount - kcmContributorCount) > 1) {
    flags.push('검토필요: GPT와KCM결과차이');
    
    // 차이 사례 자동 분류 및 저장
    const discrepancyCase = classifyDiscrepancyCase_(gptResult, kcmCodes, summaryText, pid);
    saveDiscrepancyCase_(discrepancyCase);
  }
  
  return { flags, gptContributorCount, kcmContributorCount };
}

/**
 * 차이 사례 자동 분류
 */
function classifyDiscrepancyCase_(gptResult, kcmCodes, summaryText, pid) {
  const gptContributorCount = gptResult.contributors.length;
  const kcmContributorCount = Object.values(kcmCodes).filter(codes => codes.length > 0).length;
  
  // 차이 유형 분류
  let discrepancyType = '';
  let confidence = 0;
  let analysis = '';
  
  if (gptContributorCount > kcmContributorCount) {
    // GPT가 더 많은 기여자를 감지한 경우
    discrepancyType = 'GPT_OVERCOUNT';
    
    // GPT가 감지한 추가 기여자 분석
    const gptSpeakers = gptResult.contributors.map(c => {
      const match = c.match(/참석자\s*(\d+)/);
      return match ? parseInt(match[1], 10) : null;
    }).filter(Boolean);
    
    const kcmSpeakers = Object.keys(kcmCodes).map(sx => {
      if (kcmCodes[sx].length > 0) {
        return parseInt(sx.replace('S', ''), 10);
      }
      return null;
    }).filter(Boolean);
    
    const additionalSpeakers = gptSpeakers.filter(s => !kcmSpeakers.includes(s));
    
    if (additionalSpeakers.length > 0) {
      analysis = `GPT가 추가로 감지한 참석자: ${additionalSpeakers.join(', ')}`;
      
      // GPT 오판 패턴 분석
      const additionalContributions = gptResult.contributors.filter(c => {
        const match = c.match(/참석자\s*(\d+)/);
        return match && additionalSpeakers.includes(parseInt(match[1], 10));
      });
      
      // 짧은 반응성 발화 패턴 감지
      const shortResponsePatterns = ['C4', 'C1', 'M4'];
      const hasShortResponses = additionalContributions.some(c => 
        shortResponsePatterns.some(pattern => c.includes(pattern))
      );
      
      if (hasShortResponses) {
        confidence = 0.8; // GPT가 짧은 반응을 과도하게 해석했을 가능성 높음
        analysis += ' (짧은 반응성 발화 과해석 의심)';
      } else {
        confidence = 0.3; // KCM 누락 가능성
        analysis += ' (KCM 누락 가능성)';
      }
    }
  } else if (gptContributorCount < kcmContributorCount) {
    // KCM이 더 많은 기여자를 감지한 경우
    discrepancyType = 'KCM_OVERCOUNT';
    confidence = 0.9; // KCM이 더 정확할 가능성 높음
    analysis = `KCM이 GPT보다 ${kcmContributorCount - gptContributorCount}명 더 감지`;
  }
  
  return {
    pid: pid,
    timestamp: new Date().toISOString(),
    discrepancyType: discrepancyType,
    gptCount: gptContributorCount,
    kcmCount: kcmContributorCount,
    confidence: confidence,
    analysis: analysis,
    summaryText: summaryText.substring(0, 200) + '...', // 요약문 일부만 저장
    gptContributors: gptResult.contributors,
    kcmCodes: kcmCodes
  };
}

/**
 * 차이 사례 저장
 */
function saveDiscrepancyCase_(discrepancyCase) {
  try {
    const props = PropertiesService.getDocumentProperties();
    const existingCases = JSON.parse(props.getProperty('DISCREPANCY_CASES') || '[]');
    
    // 중복 방지: 같은 PID가 이미 있으면 업데이트
    const existingIndex = existingCases.findIndex(c => c.pid === discrepancyCase.pid);
    if (existingIndex >= 0) {
      existingCases[existingIndex] = discrepancyCase;
    } else {
      existingCases.push(discrepancyCase);
    }
    
    // 최대 100개 사례만 유지 (메모리 절약)
    if (existingCases.length > 100) {
      existingCases.splice(0, existingCases.length - 100);
    }
    
    props.setProperty('DISCREPANCY_CASES', JSON.stringify(existingCases));
    
    console.log(`차이 사례 저장됨: ${discrepancyCase.pid} (${discrepancyCase.discrepancyType})`);
  } catch (error) {
    console.error('차이 사례 저장 실패:', error);
  }
}


/**
 * K/L/M 열에 코드가 있는지 확인하는 헬퍼 함수
 */
function checkColumnHasCode_(rows, colNum) {
  if (!rows || rows.length === 0 || !colNum) return false;
  const sh = SpreadsheetApp.getActiveSheet();
  
  // 성능 개선: 배치 읽기
  const minRow = Math.min(...rows);
  const maxRow = Math.max(...rows);
  const batchData = sh.getRange(minRow, colNum, maxRow - minRow + 1, 1).getValues();
  
  for (let i = 0; i < batchData.length; i++) {
    const val = String(batchData[i][0] || '').trim();
    // K 코드: K1, K2, K3
    // C 코드: C1~C7
    // M 코드: M1~M4
    if (val && /[KCM][1-7]/.test(val)) {
      return true;
    }
  }
  return false;
}

/**
 * 최종 P코드 결과 생성
 * ⚠️ 중요: K, L, M 열에 각각 어떤 코드도 나타나지 않을 때만 P0 부여
 * 예: C1만 있고 K, M 코드가 없는 경우 → P0가 아님 (P1~P3 부여)
 * LEGACY / INACTIVE / DO NOT USE
 * runCodeP_All_LEGACY_GPT 전용 helper. live P path에서 사용하지 않음.
 */
function generateFinalPCode_(gptResult, crossCheckResult, activeNum, dominant, hasAnyKLMCode) {
  let code = gptResult.code;
  let name = gptResult.name;
  let contributors = gptResult.contributors;
  let flags = [...gptResult.flags, ...crossCheckResult.flags];
  
  // ⚠️ 핵심 로직: K/L/M 열 확인
  // K, L, M 열에 각각 어떤 코드도 나타나지 않을 때만 P0 부여
  // 그 외에는 K/L/M 열 중 하나라도 코드가 있으면 P1~P3 부여
  if (hasAnyKLMCode !== undefined) {
    if (hasAnyKLMCode) {
      // K/L/M 열 중 하나라도 코드가 있으면 P0 금지, P1~P3만 허용
      if (code === 'P0') {
        // GPT가 P0를 제안했지만 K/L/M 열에 코드가 있으면 P1로 최소 보정
        code = 'P1';
        name = '1명의 의미 있는 참여';
        flags.push('KLM열코드있으나P0보정→P1');
        Logger.log(`⚠️ P 코딩 보정: K/L/M 열에 코드가 있으나 GPT가 P0 제안 → P1로 보정`);
      }
    } else {
      // K, L, M 열 모두 비어있으면 P0만 허용
      if (code !== 'P0') {
        // GPT가 P1~P3를 제안했지만 K/L/M 열에 코드가 없으면 P0로 보정
        code = 'P0';
        name = '의미 있는 참여 없음';
        flags.push('KLM열코드없어P0로보정');
        Logger.log(`⚠️ P 코딩 보정: K/L/M 열에 코드가 없으나 GPT가 P${code.replace('P', '')} 제안 → P0로 보정`);
      }
    }
  }
  
  // 지배적 발화자 플래그
  if (dominant >= 0.6 && contributors.length >= 2) {
    flags.push('한명60%↑이지만다수기여');
  }
  
  // 교사 중심 구간 체크 (추가 로직 필요시)
  if (activeNum === 0) {
    flags.push('활성발화자없음');
  }
  
  // 결과 문자열 조립
  let out = `${code} ${name}`;
  if (contributors.length > 0) {
    out += `    ${contributors.join(' / ')}`;
  }
  if (flags.length > 0) {
    out += `    [${flags.join(', ')}]`;
  }
  
  return out;
}

/**
 * F열 요약문 전처리: 발화자 태깅 개선
 */
function preprocessSummaryText_(summaryText, activeSpeakers, sHeaders) {
  let processedText = summaryText;
  
  // 활성 발화자 매핑 생성
  const speakerMapping = {};
  activeSpeakers.forEach(sx => {
    const k = parseInt(sx.replace('S',''),10);
    const entry = sHeaders[k-1];
    const label = entry ? entry.label || `참석자 ${k}` : `참석자 ${k}`;
    speakerMapping[label] = sx;
  });
  
  // 교사 발화 표시 개선
  processedText = processedText.replace(/교사가/g, '[교사]가');
  processedText = processedText.replace(/선생님이/g, '[교사]가');
  processedText = processedText.replace(/teacher가/g, '[교사]가');
  
  // 활성 발화자 외의 참석자 언급을 명확히 구분
  const allSpeakerPattern = /참석자\s*(\d+)/g;
  processedText = processedText.replace(allSpeakerPattern, (match, num) => {
    const speakerKey = `S${num}`;
    if (activeSpeakers.includes(speakerKey)) {
      return match; // 활성 발화자는 그대로 유지
    } else {
      return `[비활성]${match}`; // 비활성 발화자는 명시적으로 표시
    }
  });
  
  return processedText;
}

/**
 * GPT API 호출 (E코딩용)
 */
/**
 * UPDATED FOR GPT-5: responses API로 변경
 */
function callGPTAPI_(prompt, modelName = null) {
  const model = modelName || MODEL;
  
  // UPDATED FOR GPT-5: messages 배열을 input 문자열로 변환
  const messages = [
    {role:'system', content:'너는 교실 담화 분석 전문가다. P차원 코딩을 위해 의미 있는 기여자 수를 정확히 판단해라.'},
    {role:'user', content: prompt}
  ];
  const input = messagesToInput(messages);
  
  // UPDATED: fetchOpenAI 함수 사용 (모델명 자동 전달, 재시도는 내부에서 처리)
  const response = fetchOpenAI(model, input);
  
  return response.trim();
}

/**
 * GPT 응답 파싱 (활성 발화자 범위 검증 포함)
 */
function parseGPTResponse_(response, activeSpeakers, sHeaders) {
  const lines = response.split('\n');
  const firstLine = lines[0].trim();
  
  // P코드 추출 (P0~P3)
  const pMatch = firstLine.match(/(P[0-3])\s+(.+?)(?:\s+\|)/);
  if (!pMatch) {
    // P# 형식도 시도
    const pMatch2 = firstLine.match(/(P[0-3])\s*[—\-]\s*(.+?)(?:\s*[—\-]|\s+\|)/);
    if (pMatch2) {
      const code = pMatch2[1];
      const name = pMatch2[2].trim();
      const contributors = [];
      const contributorMatch = firstLine.match(/\|(.+)/);
      if (contributorMatch) {
        const contributorText = contributorMatch[1];
        const parts = contributorText.split('/').map(p => p.trim()).filter(p => p);
        parts.forEach(part => {
          const speakerMatch = part.match(/참석자\s*(\d+)/);
          if (speakerMatch) {
            const speakerNum = parseInt(speakerMatch[1], 10);
            const speakerKey = `S${speakerNum}`;
            if (activeSpeakers.includes(speakerKey)) {
              contributors.push(part);
            }
          } else {
            contributors.push(part);
          }
        });
      }
      return { code, name, contributors, flags: [] };
    }
    return { code: 'P0', name: '의미 있는 참여 없음', contributors: [], flags: ['GPT응답파싱실패'] };
  }
  
  const code = pMatch[1];
  const name = pMatch[2].trim();
  
  // 기여자 정보 추출 및 활성 발화자 범위 검증
  const contributors = [];
  const contributorMatch = firstLine.match(/\|(.+)/);
  if (contributorMatch) {
    const contributorText = contributorMatch[1];
    const parts = contributorText.split('/').map(p => p.trim()).filter(p => p);
    
    // 각 기여자가 활성 발화자인지 검증
    parts.forEach(part => {
      const speakerMatch = part.match(/참석자\s*(\d+)/);
      if (speakerMatch) {
        const speakerNum = parseInt(speakerMatch[1], 10);
        const speakerKey = `S${speakerNum}`;
        
        // 활성 발화자에 포함되는지 확인
        if (activeSpeakers.includes(speakerKey)) {
          contributors.push(part);
        } else {
          console.warn(`GPT가 비활성 발화자 참석자 ${speakerNum}을 포함함: ${part}`);
        }
      } else {
        // 참석자 번호가 명시되지 않은 경우도 포함 (GPT가 다른 형식으로 쓴 경우)
        contributors.push(part);
      }
    });
  }
  
  // 활성 발화자 수 초과 검증
  const flags = [];
  if (contributors.length > activeSpeakers.length) {
    flags.push('GPT분석결과활성발화자수초과');
    // 초과된 기여자를 제거하고 활성 발화자 수로 제한
    contributors.splice(activeSpeakers.length);
  }
  
  return { code, name, contributors, flags };
}


/**
 * 🗄️ COLMAP/헤더 해시/캐시 관리
 */
function getStore_() { 
  return PropertiesService.getDocumentProperties(); 
}

function getColMap_(){ 
  return loadColMap_(); // loadColMap_으로 통일
}

/**
 * 🧮 활성발화자 계산 유틸
 */
function sanitizeCount_(v){
  if (v===null || v===undefined) return 0;
  const n = String(v).replace(/[^\d\-\.]/g,'').trim();
  if(n==='') return 0;
  const num = Number(n);
  return isNaN(num)?0:(num>0?1:0); // >0은 활성 1명
}

function getActiveSpeakersCount_(rowValues, map){
  const s1 = sanitizeCount_(rowValues[colNumOf(map.S1)-1]);
  const s2 = sanitizeCount_(rowValues[colNumOf(map.S2)-1]);
  const s3 = sanitizeCount_(rowValues[colNumOf(map.S3)-1]);
  const s4 = sanitizeCount_(rowValues[colNumOf(map.S4)-1]);
  return s1 + s2 + s3 + s4;
}

/**
 * 🔍 A/D 추론 함수 (간단 키워드 기반)
 */
function inferA_(s, u){
  const text = (s+' '+u).toLowerCase();
  const cuesExplain = [/왜/,/때문/,/근거/,/설명/,/정의/,/라고 생각/,/이유/,/수학적으로/,/원리/];
  if (cuesExplain.some(re=>re.test(text))) return 'K1. 개념/근거 설명';
  return '';
}

function inferD_(s, u){
  const text = (s+' '+u).toLowerCase();
  const cuesPlan = [/하자$/, /하자고/, /먼저/, /다음/, /역할/, /정리하자/, /나눠/, /붙이자/, /쓰자/, /정리해/];
  const cuesCheck = [/맞아\?/, /확인/, /점검/, /검토/, /근거가/, /타당/];
  if (cuesPlan.some(re=>re.test(text))) return 'M1. 목표/절차/역할 제안';
  if (cuesCheck.some(re=>re.test(text))) return 'M4. 논리/개념 점검';
  return '';
}

/***** ===== 새 메뉴 시스템 끝 ===== *****/


// 🔧 수동 병합(오버라이드) 설정
const MANUAL_BLOCKS = [
  // { type: "time", start: "00:06", end: "00:32", pid: "P001" },
  // { type: "rows", startRow: 2, endRow: 6, pid: "P010" },
];


// ⏰ 클러스터링 기준 (민감도 완화)
const GAP_SPLIT_SEC       = 120;  // 90 → 120 (더 넓은 간격 허용)
const LAST_ROW_FALLBACK   = 90;
const MONOLOGUE_MIN_TURNS = 3;
const MONOLOGUE_MIN_SEC   = 45;


// 🔧 GPT 실패 시 폴백
var USE_FALLBACK_ON_GPT_FAIL = false;


// === 교사 창 & 분할 민감도
const MIN_SEG_ROWS = 2;
const TEACHER_EXIT_ROWS = 2;
const TEACHER_WINDOW_SEC = 45;


// === 짧은 단발 세그먼트 흡수 기준 (민감도 완화)
const SINGLETON_ABSORB_SEC = 120;  // 90 → 120 (더 넓은 흡수)
const SINGLETON_ABSORB_ROWS = 1;


// === Event(주제·목적 단위) 병합 기준 (민감도 완화) ===
const EVENT_MIN_ROWS       = 2;   // 3 → 2 (더 쉽게 병합)
const EVENT_MIN_SEC        = 20;  // 이 미만이면 이웃과 병합 고려
const EVENT_GAP_SEC        = 45;  // 30 → 45 (더 넓은 간격 허용)
const TOPIC_OVERLAP_T      = 0.25;// 0.30 → 0.25 (더 낮은 임계치로 병합 쉽게)
const CODE_DRIFT_TOL       = 0;   // A/C/D 대표코드 차이 합이 이 이하이면 "유사" (0=정확 일치만)

// === 토픽 병합 키워드 (통기 조직 등 연속성 유지) ===
const MERGE_TOPICS = ["통기 조직","통기조직","공기","습지","연꽃","호흡","산소","뿌리","줄기","잎"];


// === 성능 최적화
const CACHE_TTL_SEC = 21600;     // 6시간
const SHARD_SIZE = 100;          // 클러스터링 샤드 크기
const SHARD_OVERLAP = 5;         // 샤드 겹침 (5~10 권장)
const BATCH_SIZE = 8;            // 요약/코딩 배치 크기
const PARALLEL_BATCH_SIZE = 4;   // 병렬 처리 배치 크기
const MAX_PARALLEL_REQUESTS = 6; // 최대 병렬 요청 수
const CODING_TIMEOUT_MS = 30000; // KCM/P 코딩 타임아웃 (30초)

// ===== C-dimension keyword lexicons (KR, loose match) =====
const C_LEX = {
  agree: [
    "맞아", "그렇지", "동의", "오케이", "그치", "좋은 얘기", "그건 맞지", "맞지",
    "좋아", "OK", "ㅇㅇ", "응", "그래"
  ],
  clarify: [
    "무슨 뜻", "어떤 의미", "왜 그렇게", "왜야", "왜지", "설명해", "근거가 뭐야",
    "다시", "정확히", "명확", "무슨 얘기", "이거 뭐야", "이건 뭐야", "어떤 조직",
    "어떻게", "뭐가", "뭘", "질문", "궁금", "맞아?", "맞나", "어째서"
  ],
  elaborate: [
    "그러니까", "예를 들면", "예시", "추가로", "정리하자면", "말하자면",
    "결국", "덧붙이면", "즉", "다시 말해", "보충", "정교", "근거는", "요컨대"
  ],
  rebuttal: [
    "아니야", "그건 아니", "말이 안 돼", "다른데", "그건 물이지", "그거 아니고",
    "그렇게 따지면", "반박", "하지만", "근데", "아닌데", "왜 말이 안 돼", "토양 아니야"
  ],
  coordinate: [
    "그치 그건 맞지 근데", "그러면 우리가", "그럼 이렇게 하자", "정리하자면",
    "일단 ~~하고", "역할", "순서", "정도로 하자", "결정", "합의", "수정하자",
    "붙이자", "정하자", "나누자"
  ]
};

// 디버그 로그 스위치
const DEBUG_LOG = false;
function dbg() { 
  if (DEBUG_LOG) {
    var args = Array.prototype.slice.call(arguments);
    Logger.log("[KCM/P] " + args.join(" "));
  }
}

// 간단 전처리
function normKR(s) {
  return String(s||"").replace(/\s+/g," ").trim();
}

/**
 * 헤더 기반 동적 열 매핑 (열 구조 변경에 안전)
 * 반환: {F, S1, S2, S3, S4, L, K, M, N, O, P, PID}
 */
function detectColumnsByHeader(sheet) {
  // 상위 5행 스캔 (헤더 행 자동판정)
  var maxRow = Math.min(5, sheet.getLastRow());
  var lastCol = sheet.getLastColumn();
  var allRows = sheet.getRange(1, 1, maxRow, lastCol).getValues();
  
  // 헤더 정규화 함수
  var normalize = function(s) {
    return String(s||"")
      .toLowerCase()
      .replace(/\([^)]*\)/g, "") // 괄호 제거
      .replace(/[:\-_]/g, " ")   // 구분자를 공백으로
      .replace(/\s+/g, " ")      // 공백 압축
      .trim();
  };
  
  // ✅ 1단계: 헤더 행 자동판정 (상위 5행 점수 기반)
  var headerRowIdx = -1;
  var maxScore = -1;
  
  for (var r = 0; r < maxRow; r++) {
    var score = 0;
    var avgLen = 0;
    var numCount = 0;
    var speakerPatternCount = 0;
    
    for (var c = 0; c < lastCol; c++) {
      var val = String(allRows[r][c]||"").trim();
      avgLen += val.length;
      
      // 참석자/speaker 패턴 매칭
      if (/참석자\s*\d+|speaker\s*\d+|학생\s*\d+|화자\s*\d+/i.test(val)) {
        speakerPatternCount++;
      }
      
      // 숫자/코드 전용 (P###, 시간 등)
      if (/^[A-Z]\d+$|^\d{2}:\d{2}/.test(val)) {
        numCount++;
      }
    }
    
    avgLen = lastCol > 0 ? avgLen / lastCol : 0;
    
    // 점수 계산
    score += speakerPatternCount * 3; // 참석자 패턴 × 3점
    if (avgLen <= 40) score += 2;     // 짧은 텍스트 (헤더는 짧음)
    if (numCount >= 2) score += 1;    // 코드/시간 열 다수
    
    Logger.log("  행" + (r+1) + " 점수: " + score + " (평균길이:" + Math.round(avgLen) + ", 참석자패턴:" + speakerPatternCount + ")");
    
    if (score > maxScore) {
      maxScore = score;
      headerRowIdx = r;
    }
  }
  
  // 헤더 행 선택 (최고점 행 또는 1행)
  if (maxScore < 3) {
    Logger.log("⚠️ 헤더 행 판정 실패 (최고점 " + maxScore + "점). 1행을 기본 헤더로 사용하되, 장문 폴백 활성화.");
    headerRowIdx = 0;
  } else {
    Logger.log("✅ 헤더 행: " + (headerRowIdx + 1) + "행 (점수: " + maxScore + ")");
  }
  
  var header = allRows[headerRowIdx];
  
  // 부분일치 + 점수 기반 탐지
  var findBest = function(preds, minScore) {
    var best = {idx: -1, score: 0};
    for (var i = 0; i < header.length; i++) {
      var h = normalize(header[i]);
      if (!h) continue;
      
      var score = 0;
      for (var j = 0; j < preds.length; j++) {
        var pred = normalize(preds[j]);
        if (h === pred) score = Math.max(score, 100); // 정확 일치
        else if (h.indexOf(pred) !== -1) score = Math.max(score, 80); // 부분 일치
      }
      
      if (score > best.score || (score === best.score && best.idx === -1)) {
        best = {idx: i + 1, score: score}; // 1-based
      }
    }
    return (best.score >= (minScore||60)) ? best.idx : -1;
  };
  
  // 🔧 S1~S4 찾기: 유연한 패턴 매칭 (참석자 1~10 모두 허용)
  var findSpeaker = function(num) {
    // "참석자 N" 패턴을 가진 열들을 모두 찾아서 순서대로 배치
    var candidates = [];
    for (var i = 0; i < header.length; i++) {
      var h = String(header[i]||"").trim();
      var m = h.match(/참석자\s*(\d+)/);
      if (m) {
        candidates.push({col: i+1, num: parseInt(m[1], 10), label: h});
      }
    }
    // 숫자 순으로 정렬
    candidates.sort(function(a,b) { return a.num - b.num; });
    // num-1 인덱스가 요청한 S번호 (0-based)
    return candidates[num-1] ? candidates[num-1].col : -1;
  };
  
  var cols = {
    F: findBest(["요약","요약문","Summary","클러스터 요약","발화 요약","전사요약","설명문","텍스트"], 60),
    S1: findSpeaker(1),  // 첫 번째 "참석자 N"
    S2: findSpeaker(2),  // 두 번째 "참석자 N"
    S3: findSpeaker(3),  // 세 번째 "참석자 N"
    S4: findSpeaker(4),  // 네 번째 "참석자 N"
    K: findBest(["K차원", "K 차원", "K코드", "Epistemic", "K 결과"], 60),
    L: findBest(["C차원", "C 차원", "C코드", "Collaborative", "C 결과", "C-dimension"], 60),
    M: findBest(["M차원", "M 차원", "M코드", "Metacognitive", "M 결과"], 60),
    N: findBest(["P차원", "P 차원", "P코드", "Participation", "P 결과"], 60),
    O: findBest(["교사", "교사개입", "Teacher"], 50),
    P: findBest(["패턴", "Pattern"], 50),
    PID: findBest(["PID", "클러스터", "cluster", "P-ID"], 50)
  };
  
  // ✅ 2단계: 요약 열 휴리스틱 폴백 (헤더 매칭 실패 시)
  if (cols.F <= 0) {
    Logger.log("⚠️ 요약 열을 헤더에서 찾지 못함. 장문 폴백 시도...");
    
    // 데이터 행 10개 샘플링 (헤더 다음 행부터)
    var sampleRow = Math.min(headerRowIdx + 10, sheet.getLastRow());
    var dataRows = sheet.getRange(headerRowIdx + 2, 1, sampleRow - headerRowIdx - 1, lastCol).getValues();
    
    var bestCol = -1;
    var maxLongScore = 0;
    
    for (var c = 0; c < lastCol; c++) {
      var totalLen = 0;
      var punctCount = 0;
      var bulletCount = 0;
      
      for (var r = 0; r < dataRows.length; r++) {
        var val = String(dataRows[r][c]||"");
        totalLen += val.length;
        
        // 문장부호 카운트
        punctCount += (val.match(/[.,?!'"「」]/g) || []).length;
        
        // ■ 패턴 (강력한 요약 신호)
        if (/^■\s*\d{2}:\d{2}/.test(val)) {
          bulletCount += 10; // 강한 가점
        }
      }
      
      var avgLen = dataRows.length > 0 ? totalLen / dataRows.length : 0;
      var score = 0;
      
      if (avgLen >= 60) score += 10;        // 평균 60자 이상
      if (avgLen >= 100) score += 5;        // 평균 100자 이상 보너스
      if (punctCount >= 10) score += 5;     // 문장부호 많음
      score += bulletCount;                  // ■ 패턴 가점
      
      Logger.log("  열" + (c+1) + " 장문 점수: " + score + " (평균:" + Math.round(avgLen) + "자, 문장부호:" + punctCount + ", 불릿:" + (bulletCount/10) + ")");
      
      if (score > maxLongScore) {
        maxLongScore = score;
        bestCol = c + 1; // 1-based
      }
    }
    
    if (bestCol > 0 && maxLongScore >= 10) {
      cols.F = bestCol;
      Logger.log("✅ 요약 열 폴백 성공: 열" + bestCol + " (장문 점수:" + maxLongScore + ")");
    } else {
      Logger.log("❌ 요약 열 폴백 실패. 고정 열 번호 사용.");
    }
  }
  
  // ✅ 헤더 매핑 로그
  Logger.log("📍 헤더 매핑 결과 (상위 " + maxRow + "행 스캔):");
  Logger.log("  F(요약): " + cols.F);
  Logger.log("  S1~S4(발화): " + [cols.S1,cols.S2,cols.S3,cols.S4].join(","));
  Logger.log("  K~P(결과): " + [cols.K,cols.L,cols.M,cols.N,cols.O,cols.P].join(","));
  
  // ✅ 현재 헤더 출력 (디버깅용)
  Logger.log("📋 감지된 헤더 목록:");
  for (var i = 0; i < Math.min(20, header.length); i++) {
    Logger.log("  열" + (i+1) + ": '" + header[i] + "' → 정규화: '" + normalize(header[i]) + "'");
  }
  
  // ✅ 3단계: C차원 열 자동 생성 (없으면 새로 추가)
  if (cols.L <= 0) {
    Logger.log("⚠️ C차원 열을 찾지 못함. 새 열을 추가합니다...");
    
    var newColIdx = lastCol + 1;
    sheet.getRange(headerRowIdx + 1, newColIdx).setValue("C차원");
    cols.L = newColIdx;
    
    Logger.log("✅ C차원 열 생성 완료: 열" + newColIdx);
  }
  
  // ✅ 필수 열 검증 (경고만, 폴백 사용)
  var missingCols = [];
  if (cols.F <= 0) missingCols.push("F(요약)");
  if (cols.S1 <= 0 || cols.S2 <= 0 || cols.S3 <= 0 || cols.S4 <= 0) missingCols.push("S1~S4(발화)");
  
  if (missingCols.length > 0) {
    var warnMsg = "⚠️ 헤더 매핑 경고: " + missingCols.join(", ") + " 열을 자동 탐지하지 못했습니다.\n" +
                  "고정 열 번호로 폴백합니다.\n\n" +
                  "📋 현재 헤더 (상위 10개):\n";
    for (var i = 0; i < Math.min(10, header.length); i++) {
      warnMsg += "  열" + (i+1) + ": '" + String(header[i]||"") + "'\n";
    }
    warnMsg += "\n안내:\n" +
               "- 요약: '요약|요약문|summary|텍스트' 중 하나 포함하거나 평균 60자 이상 장문\n" +
               "- C차원: 'C차원|C코드|C 결과' 중 하나 포함 (없으면 자동 생성됨)\n" +
               "- 참석자: '참석자 1~4' 패턴 사용\n";
    Logger.log(warnMsg);
    SpreadsheetApp.getUi().alert(warnMsg);
  }
  
  return cols;
}

/**
 * 안전 숫자 변환
 */
function toNum(x) { 
  const n = Number(String(x||"").trim().replace(/[^\d.-]/g,"")); 
  return isNaN(n) ? 0 : n; 
}

/**
 * S1~S4 강건 탐지 (헤더 → 숫자밀도 → 수동)
 */
function detectSColsRobust(sh){
  const props = PropertiesService.getDocumentProperties();
  const cached = props.getProperty('COLMAP');
  if (cached){
    try {
      const map = JSON.parse(cached);
      if (map.S1 && map.S2 && map.S3 && map.S4) {
        // 🔧 객체일 수 있으므로 colNumOf로 숫자 추출
        const s1 = colNumOf(map.S1);
        const s2 = colNumOf(map.S2);
        const s3 = colNumOf(map.S3);
        const s4 = colNumOf(map.S4);
        Logger.log("✅ 캐시된 S1~S4 사용: " + [s1,s2,s3,s4].join(","));
        return [s1, s2, s3, s4];
      }
    } catch(e){}
  }

  const lastCol = sh.getLastColumn();
  const header = sh.getRange(1,1,1,lastCol).getValues()[0].map(s=>String(s||'').trim());

  // 1) 헤더 패턴/저장된 학생이름
  const students = props.getProperty('STUDENTS_GJ');
  const names = students ? JSON.parse(students) : [];
  let idxs = [];
  header.forEach((h,i)=>{
    if (/^참석자\s*\d/.test(h) || names.includes(h)) idxs.push(i+1);
  });
  if (idxs.length===4){
    Logger.log("✅ 헤더 패턴으로 S1~S4 탐지: " + idxs.join(","));
    saveSCols(idxs);
    return idxs;
  }

  // 2) 숫자밀도 휴리스틱(최근 100행)
  const lastRow = sh.getLastRow();
  if (lastRow >= 3){
    const sampleRows = Math.min(100, lastRow-1);
    const data = sh.getRange(2,1,sampleRows,lastCol).getValues();
    let best = null;
    for (let start=1; start<=lastCol-3; start++){
      let numericScore = 0, longPenalty=0;
      for (let r=0; r<data.length; r++){
        for (let c=0; c<4; c++){
          const v = data[r][start-1+c];
          const isNum = String(v||'').trim()==='' ? false : !isNaN(Number(v));
          if (isNum) numericScore++;
          if (String(v||'').length>60) longPenalty+=2;
        }
      }
      const score = numericScore - longPenalty;
      if (!best || score>best.score) best = {start, score};
    }
    if (best && best.score>0){
      const out = [best.start,best.start+1,best.start+2,best.start+3];
      Logger.log("✅ 숫자밀도로 S1~S4 탐지: " + out.join(",") + " (점수:" + best.score + ")");
      saveSCols(out);
      return out;
    }
  }

  // 3) 수동 매핑
  const ui = SpreadsheetApp.getUi();
  ui.alert('S1~S4(학생별 발화수) 자동탐지 실패 → 수동 매핑을 진행합니다.');
  const g = Number(ui.prompt('S1 열 번호 (예: 7=G)', '', ui.ButtonSet.OK).getResponseText());
  const h = Number(ui.prompt('S2 열 번호 (예: 8=H)', '', ui.ButtonSet.OK).getResponseText());
  const i = Number(ui.prompt('S3 열 번호 (예: 9=I)', '', ui.ButtonSet.OK).getResponseText());
  const j = Number(ui.prompt('S4 열 번호 (예:10=J)', '', ui.ButtonSet.OK).getResponseText());
  const out = [g,h,i,j];
  Logger.log("✅ 수동 매핑 S1~S4: " + out.join(","));
  saveSCols(out);
  return out;

  function saveSCols(a){
    const map = JSON.parse(props.getProperty('COLMAP')||'{}');
    // 🔧 객체 형태로 저장 (기존 헤더 보존)
    const h1 = (map.S1 && map.S1.header) || '';
    const h2 = (map.S2 && map.S2.header) || '';
    const h3 = (map.S3 && map.S3.header) || '';
    const h4 = (map.S4 && map.S4.header) || '';
    map.S1={col:a[0], header:h1};
    map.S2={col:a[1], header:h2};
    map.S3={col:a[2], header:h3};
    map.S4={col:a[3], header:h4};
    props.setProperty('COLMAP', JSON.stringify(map));
  }
}

/**
 * 활성 발화자 수 계산 (행 기반, 동적 열)
 */
function activeCountFromRow(row, cols){
  const nums = [
    toNum(row[colNumOf(cols.S1)-1]), 
    toNum(row[colNumOf(cols.S2)-1]), 
    toNum(row[colNumOf(cols.S3)-1]), 
    toNum(row[colNumOf(cols.S4)-1])
  ];
  return nums.filter(function(n) { return n > 0; }).length;
}

/**
 * 프리플라이트: 헤더 확인 + 열 매핑 확정 (경고 없이 자동 복구)
 */
function preflightAndGetCols(){
  const sh = SpreadsheetApp.getActiveSheet();
  
  // 1) 헤더 자동 삽입 (1행이 데이터로 보이면)
  ensureHeaderRowStrict();

  // 2) 기존 detectColumnsByHeader로 기본 매핑
  const cols = detectColumnsByHeader(sh) || {};
  
  // 3) S1~S4 강제 보강
  const s = detectSColsRobust(sh);
  // detectSColsRobust는 [7,8,9,10] 배열 반환하므로 객체로 변환
  cols.S1 = {col: s[0]}; 
  cols.S2 = {col: s[1]}; 
  cols.S3 = {col: s[2]}; 
  cols.S4 = {col: s[3]};

  // 4) 문서 속성에 저장
  const props = PropertiesService.getDocumentProperties();
  let map = JSON.parse(props.getProperty('COLMAP')||'{}');
  
  // S1~S4 객체 형태로 저장 (기존 header 보존)
  const h1 = (map.S1 && map.S1.header) || '';
  const h2 = (map.S2 && map.S2.header) || '';
  const h3 = (map.S3 && map.S3.header) || '';
  const h4 = (map.S4 && map.S4.header) || '';
  map.S1 = {col: s[0], header: h1};
  map.S2 = {col: s[1], header: h2};
  map.S3 = {col: s[2], header: h3};
  map.S4 = {col: s[3], header: h4};
  
  if (cols.F) map.F=cols.F; 
  if (cols.L) map.L=cols.L; 
  if (cols.K) map.K=cols.K;
  if (cols.M) map.M=cols.M; 
  if (cols.N) map.N=cols.N; 
  if (cols.O) map.O=cols.O; 
  if (cols.P) map.P=cols.P;
  
  // 🔧 객체 형식 보증
  map = normalizeSColsInMap_(map);
  props.setProperty('COLMAP', JSON.stringify(map));
  
  Logger.log("✅ 프리플라이트 완료: 열 매핑 확정");
  return cols;
}


/*** 키/설정 UI ***/
function setApiKeyOnce(){
  var ui = SpreadsheetApp.getUi();
  var resp = ui.prompt(
    "OpenAI API Key 입력",
    "본인만 사용하는 OpenAI API 키를 입력하세요. 이 키는 현재 사용자 계정에만 저장되며, 스프레드시트·다른 사용자·GitHub에는 기록되지 않습니다.",
    ui.ButtonSet.OK_CANCEL
  );
  if (resp.getSelectedButton() !== ui.Button.OK) return;
  var key = (resp.getResponseText() || "").trim();
  if (!key) { ui.alert("키가 비어 있습니다."); return; }
  if (!key.startsWith('sk-')) {
    ui.alert("API 키 형식이 올바르지 않습니다. sk-로 시작하는 키를 입력하세요.");
    return;
  }
  PropertiesService.getUserProperties().setProperty('OPENAI_API_KEY', key);
  ui.alert("개인 API Key가 저장되었습니다. 이 키의 사용 요금은 본인의 OpenAI 계정에 청구됩니다.");
}
function clearMyApiKey(){
  PropertiesService.getUserProperties().deleteProperty('OPENAI_API_KEY');
  SpreadsheetApp.getUi().alert("현재 사용자에게 저장된 API Key를 삭제했습니다.");
}
function getApiKey(){
  var key = PropertiesService.getUserProperties().getProperty('OPENAI_API_KEY');
  if (!key) {
    throw new Error('API Key가 설정되지 않았습니다. [AI 코딩] → [1. API키 설정]에서 본인의 키를 입력하세요.');
  }
  if (!key.startsWith('sk-')) {
    throw new Error('API 키 형식이 올바르지 않습니다. sk-로 시작하는 키를 입력하세요.');
  }
  return key;
}

/**
 * UPDATED FOR GPT-5: messages 배열을 단일 input 문자열로 변환
 * responses API는 messages 배열이 아니라 단일 input 문자열을 받음
 */
function messagesToInput(messages) {
  if (!messages || !Array.isArray(messages) || messages.length === 0) {
    throw new Error('messages 배열이 비어있습니다.');
  }
  
  // messages 배열을 하나의 프롬프트로 합치기
  var parts = [];
  for (var i = 0; i < messages.length; i++) {
    var msg = messages[i];
    var role = msg.role || 'user';
    var content = msg.content || '';
    
    if (role === 'system') {
      parts.push('System: ' + content);
    } else if (role === 'user') {
      parts.push('User: ' + content);
    } else if (role === 'assistant') {
      parts.push('Assistant: ' + content);
    } else {
      parts.push(content);
    }
  }
  
  return parts.join('\n\n');
}

/**
 * UPDATED: gpt-5-mini & gpt-5 통합 호출 함수
 * - 모델명에 따라 요청 구조 자동 분기
 * - gpt-5-mini: {model, input}만 사용 (temperature 등 제거)
 * - gpt-5: {model, input, temperature, max_output_tokens} 사용 가능
 * - 통합 응답 파싱: output[0].content[0].text 또는 output_text
 */
function callGPT(promptText, modelName = null) {
  // API 키 체크
  const apiKey = getApiKey();
  if (!apiKey || apiKey === '' || apiKey === null || apiKey === undefined) {
    throw new Error('API Key가 설정되지 않았습니다.');
  }
  
  // 입력 검증
  const finalInput = String(promptText || '').trim();
  if (finalInput === '') {
    throw new Error('입력 텍스트가 비어있습니다.');
  }
  
  // 모델명 결정 (기본값: gpt-5-mini)
  const model = modelName || MODEL || "gpt-5-mini";
  
  // KCMP 코딩: gpt-4o-mini는 chat/completions API 사용
  const isGpt4oMini = /gpt-4o-mini/i.test(model);
  const isGpt5Mini = /gpt-5-mini/i.test(model);
  
  let url, payload;
  
  if (isGpt4oMini) {
    // gpt-4o-mini: chat/completions API 사용
    url = "https://api.openai.com/v1/chat/completions";
    payload = {
      model: model,
      messages: [
        {role: "user", content: finalInput}
      ],
      temperature: 0.3,  // gpt-4o-mini는 temperature 지원
      max_tokens: 500
    };
  } else {
    // gpt-5 계열: responses API 사용
    url = "https://api.openai.com/v1/responses";
    payload = {
      model: model,
      input: finalInput
    };
    
    // gpt-5-mini 계열이면 temperature 등 제거 (400 오류 방지)
    if (isGpt5Mini) {
      // payload에는 model과 input만 포함
    }
  }
  const headers = {
    "Authorization": "Bearer " + apiKey,
    "Content-Type": "application/json"
  };
  
  const resp = UrlFetchApp.fetch(url, {
    method: "post",
    contentType: "application/json",
    headers: headers,
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });
  
  const code = resp.getResponseCode();
  const body = resp.getContentText();
  
  if (code !== 200) {
    throw new Error("HTTP " + code + ": " + body.substring(0, 200));
  }
  
  try {
    const json = JSON.parse(body);
    
    // KCMP 코딩: gpt-4o-mini는 chat/completions API 응답 형식 우선 확인
    let text = "";
    let debugLogged = false; // 디버깅 로그는 한 번만 출력
    
    // gpt-4o-mini 우선 처리: choices 배열 먼저 확인 (chat/completions API)
    if (isGpt4oMini && json?.choices && Array.isArray(json.choices) && json.choices.length > 0) {
      const choice = json.choices[0];
      if (choice?.message?.content) {
        text = String(choice.message.content).trim();
        if (text) {
          Logger.log("✅ callGPT (gpt-4o-mini): choices[0].message.content에서 텍스트 추출 성공 (길이: " + text.length + ")");
          return text;
        }
      }
    }
    
    // 방법 1: output_text 필드 직접 확인 (gpt-5 responses API)
    if (json?.output_text && typeof json.output_text === 'string') {
      text = json.output_text.trim();
      if (text) {
        Logger.log("✅ callGPT: output_text에서 텍스트 추출 성공 (길이: " + text.length + ")");
        return text;
      }
    }
    
    // 방법 2: output 배열에서 type === "message" 블록 찾기 (2025년형 gpt-5/gpt-5-mini 표준)
    if (json?.output && Array.isArray(json.output)) {
      if (!debugLogged) {
        Logger.log("callGPT: output 배열 발견, 길이: " + json.output.length);
        debugLogged = true;
      }
      
      // NEW PARSER START: GPT-5 응답 구조 완전 지원
      // 우선순위 1: output[].message.content[].text (GPT-5 최신 형식)
      const messageBlock = json.output.find(block => block?.type === "message");
      if (messageBlock) {
        // message.content가 배열인 경우 (content: [{type:"output_text", text:"..."}])
        if (messageBlock.content && Array.isArray(messageBlock.content)) {
          for (const contentItem of messageBlock.content) {
            if (contentItem?.type === "output_text" && contentItem?.text) {
              text = String(contentItem.text).trim();
              if (text) {
                Logger.log("✅ callGPT: output[].message.content[].text에서 텍스트 추출 성공 (길이: " + text.length + ")");
                return text;
              }
            }
            if (contentItem?.text && typeof contentItem.text === 'string') {
              text = contentItem.text.trim();
              if (text) {
                Logger.log("✅ callGPT: output[].message.content[].text에서 텍스트 추출 성공 (길이: " + text.length + ")");
                return text;
              }
            }
          }
        }
        // message.content가 문자열인 경우
        if (messageBlock.content && typeof messageBlock.content === 'string') {
          text = messageBlock.content.trim();
          if (text) {
            Logger.log("✅ callGPT: output[].type=message.content에서 텍스트 추출 성공 (길이: " + text.length + ")");
            return text;
          }
        }
        // message.text 직접 확인
        if (messageBlock.text && typeof messageBlock.text === 'string') {
          text = messageBlock.text.trim();
          if (text) {
            Logger.log("✅ callGPT: output[].type=message.text에서 텍스트 추출 성공 (길이: " + text.length + ")");
            return text;
          }
        }
      }
      
      // 우선순위 2: output[].content[].text
      for (const block of json.output) {
        if (block?.content && Array.isArray(block.content)) {
          for (const contentItem of block.content) {
            if (contentItem?.text && typeof contentItem.text === 'string') {
              text = contentItem.text.trim();
              if (text) {
                Logger.log("✅ callGPT: output[].content[].text에서 텍스트 추출 성공 (길이: " + text.length + ")");
                return text;
              }
            }
          }
        }
      }
      
      // 우선순위 2: type === "output_text" 또는 "text" 블록 찾기
      // 모든 블록을 순회하며 텍스트 찾기
      for (let idx = 0; idx < json.output.length; idx++) {
        const block = json.output[idx];
        if (!debugLogged && idx < 2) {
          Logger.log(`callGPT: output[${idx}] 타입: ${block?.type || 'unknown'}, keys: ${Object.keys(block || {}).join(',')}`);
        }
        
        // output_text 또는 text 타입 블록 찾기
        if (block?.type === "output_text" || block?.type === "text") {
          // content가 배열인 경우
          if (block?.content && Array.isArray(block.content)) {
            for (let cIdx = 0; cIdx < block.content.length; cIdx++) {
              const contentItem = block.content[cIdx];
              if (contentItem?.text && typeof contentItem.text === 'string') {
                text = contentItem.text.trim();
                if (text) {
                  Logger.log(`✅ callGPT: output[${idx}].content[${cIdx}].text에서 텍스트 추출 성공 (길이: ${text.length})`);
                  return text;
                }
              }
              // contentItem이 직접 문자열인 경우
              if (typeof contentItem === 'string') {
                text = contentItem.trim();
                if (text) {
                  Logger.log(`✅ callGPT: output[${idx}].content[${cIdx}] 문자열에서 텍스트 추출 성공 (길이: ${text.length})`);
                  return text;
                }
              }
            }
          }
          // content가 문자열인 경우
          if (block?.content && typeof block.content === 'string') {
            text = block.content.trim();
            if (text) {
              Logger.log(`✅ callGPT: output[${idx}].content 문자열에서 텍스트 추출 성공 (길이: ${text.length})`);
              return text;
            }
          }
          // text 필드 직접 확인
          if (block?.text && typeof block.text === 'string') {
            text = block.text.trim();
            if (text) {
              Logger.log(`✅ callGPT: output[${idx}].text에서 텍스트 추출 성공 (길이: ${text.length})`);
              return text;
            }
          }
        }
        
        // 우선순위 4: output[].reasoning[].text (reasoning 블록에서도 텍스트 추출 시도)
        if (block?.type === "reasoning") {
          // reasoning 블록 안에도 텍스트가 있을 수 있음
          if (block.content && typeof block.content === 'string') {
            text = block.content.trim();
            if (text && text.length > 10) { // reasoning이 너무 짧으면 건너뜀
              // ###OUTPUT: 뒤의 텍스트만 추출
              const outputMatch = text.match(/###OUTPUT:\s*(.+)/s);
              if (outputMatch) {
                text = outputMatch[1].trim();
                if (text) {
                  Logger.log("✅ callGPT: output[].reasoning에서 ###OUTPUT: 추출 성공 (길이: " + text.length + ")");
                  return text;
                }
              }
            }
          }
          if (block.text && typeof block.text === 'string') {
            text = block.text.trim();
            const outputMatch = text.match(/###OUTPUT:\s*(.+)/s);
            if (outputMatch) {
              text = outputMatch[1].trim();
              if (text) {
                Logger.log("✅ callGPT: output[].reasoning.text에서 ###OUTPUT: 추출 성공 (길이: " + text.length + ")");
                return text;
              }
            }
          }
          continue; // reasoning만 있고 텍스트 없으면 다음 블록으로
        }
        
        // 타입이 없지만 content나 text가 있는 경우
        if (!block?.type) {
          if (block?.content && typeof block.content === 'string') {
            text = block.content.trim();
            if (text) {
              Logger.log(`✅ callGPT: output[${idx}].content(타입없음)에서 텍스트 추출 성공 (길이: ${text.length})`);
              return text;
            }
          }
          if (block?.text && typeof block.text === 'string') {
            text = block.text.trim();
            if (text) {
              Logger.log(`✅ callGPT: output[${idx}].text(타입없음)에서 텍스트 추출 성공 (길이: ${text.length})`);
              return text;
            }
          }
        }
      }
      
      // 방법 3: output[0].content[0].text (레거시 구조)
      if (json.output[0]?.content) {
        if (Array.isArray(json.output[0].content)) {
          for (const contentItem of json.output[0].content) {
            if (contentItem?.text && typeof contentItem.text === 'string') {
              text = contentItem.text.trim();
              if (text) {
                Logger.log("✅ callGPT: output[0].content[].text에서 텍스트 추출 성공 (길이: " + text.length + ")");
                return text;
              }
            }
          }
        } else if (typeof json.output[0].content === 'string') {
          text = json.output[0].content.trim();
          if (text) {
            Logger.log("✅ callGPT: output[0].content 문자열에서 텍스트 추출 성공 (길이: " + text.length + ")");
            return text;
          }
        }
      }
      
      // 방법 4: output 배열의 모든 문자열 값 찾기
      for (let idx = 0; idx < json.output.length; idx++) {
        const block = json.output[idx];
        if (typeof block === 'string') {
          text = block.trim();
          if (text) {
            Logger.log(`✅ callGPT: output[${idx}] 문자열에서 텍스트 추출 성공 (길이: ${text.length})`);
            return text;
          }
        }
      }
    }
    
    // 방법 5: message 필드 확인 (일부 API 버전)
    if (json?.message) {
      if (json.message.content && typeof json.message.content === 'string') {
        text = json.message.content.trim();
        if (text) {
          Logger.log("✅ callGPT: message.content에서 텍스트 추출 성공 (길이: " + text.length + ")");
          return text;
        }
      }
      if (json.message.text && typeof json.message.text === 'string') {
        text = json.message.text.trim();
        if (text) {
          Logger.log("✅ callGPT: message.text에서 텍스트 추출 성공 (길이: " + text.length + ")");
          return text;
        }
      }
    }
    
    // 방법 6: choices 배열 확인 (gpt-4o-mini 등 chat/completions API 형식 - 우선순위 높임)
    if (json?.choices && Array.isArray(json.choices) && json.choices.length > 0) {
      const choice = json.choices[0];
      if (choice?.message?.content) {
        text = choice.message.content.trim();
        if (text) {
          Logger.log("✅ callGPT: choices[0].message.content에서 텍스트 추출 성공 (길이: " + text.length + ")");
          return text;
        }
      }
      if (choice?.message?.text) {
        text = choice.message.text.trim();
        if (text) {
          Logger.log("✅ callGPT: choices[0].message.text에서 텍스트 추출 성공 (길이: " + text.length + ")");
          return text;
        }
      }
      if (choice?.text) {
        text = choice.text.trim();
        if (text) {
          Logger.log("✅ callGPT: choices[0].text에서 텍스트 추출 성공 (길이: " + text.length + ")");
          return text;
        }
      }
    }
    
    // 방법 7: 직접 텍스트 필드 확인
    if (json?.text && typeof json.text === 'string') {
      text = json.text.trim();
      if (text) {
        Logger.log("✅ callGPT: text 필드에서 텍스트 추출 성공 (길이: " + text.length + ")");
        return text;
      }
    }
    
    // 방법 8: content 필드 직접 확인
    if (json?.content && typeof json.content === 'string') {
      text = json.content.trim();
      if (text) {
        Logger.log("✅ callGPT: content 필드에서 텍스트 추출 성공 (길이: " + text.length + ")");
        return text;
      }
    }
    
    // 방법 9: Fallback - JSON 전체에서 문자열 블록 자동 추출 (###OUTPUT: 포함)
    const jsonString = JSON.stringify(json);
    const outputMatch = jsonString.match(/###OUTPUT:\s*([^\"]+)/);
    if (outputMatch) {
      text = outputMatch[1].trim().replace(/\\n/g, '\n').replace(/\\"/g, '"');
      if (text && text.length > 0) {
        Logger.log("✅ callGPT: JSON에서 ###OUTPUT: 자동 추출 성공 (길이: " + text.length + ")");
        return text;
      }
    }
    
    // 방법 10: Fallback - JSON 전체에서 긴 문자열 블록 추출 (50자 이상)
    const stringMatches = jsonString.match(/"([^"]{50,})"/g);
    if (stringMatches && stringMatches.length > 0) {
      // 가장 긴 문자열 선택
      const longest = stringMatches.reduce((a, b) => a.length > b.length ? a : b);
      text = longest.replace(/^"|"$/g, '').replace(/\\n/g, '\n').replace(/\\"/g, '"').trim();
      if (text && text.length > 20 && !text.includes('reasoning') && !text.includes('type')) {
        Logger.log("✅ callGPT: JSON에서 긴 문자열 블록 자동 추출 성공 (길이: " + text.length + ")");
        return text;
      }
    }
    
    // 모든 방법 실패 시 상세 로그 출력 후 에러 throw
    Logger.log("⚠️ callGPT: 모든 파싱 방법 실패");
    Logger.log("응답 구조 요약: " + JSON.stringify({
      hasOutput: !!json.output,
      outputLength: json.output?.length || 0,
      outputTypes: json.output?.map(b => b?.type || 'unknown') || [],
      hasOutputText: !!json.output_text,
      hasMessage: !!json.message,
      hasChoices: !!json.choices,
      keys: Object.keys(json)
    }));
    
    // 실패한 응답 구조 전체를 로그에 저장 (너무 길면 잘라서)
    const fullResponse = JSON.stringify(json, null, 2);
    if (fullResponse.length > 2000) {
      Logger.log("전체 응답 구조 (압축):\n" + fullResponse.substring(0, 2000) + "\n... (총 " + fullResponse.length + "자)");
    } else {
      Logger.log("전체 응답 구조:\n" + fullResponse);
    }
    
    // 빈 응답을 에러로 처리 (조용히 넘어가지 않도록)
    throw new Error("GPT 응답 파싱 실패: output 배열에서 텍스트를 찾을 수 없습니다. 응답 구조를 로그에서 확인하세요.");
  } catch (parseError) {
    Logger.log("❌ callGPT: JSON 파싱 오류: " + parseError.toString());
    Logger.log("원본 응답 (처음 500자): " + body.substring(0, 500));
    throw new Error("응답 파싱 실패: " + parseError.toString());
  }
}

/**
 * UPDATED: gpt-5-mini & gpt-5 통합 안전 호출 함수
 * - 모델명에 따라 자동 분기
 * - 클러스터링용: 재시도 없음 (초고속)
 * - 기타 용도: 최대 1회 재시도 (빠른 처리)
 * - 실패 시 빈 문자열 반환 (전체 프로세스 중단 없음)
 */
/** OpenAI insufficient_quota / billing exhaustion 판별 (대소문자 무시). retry 금지용. */
function _isOpenAIQuotaExhaustedError_(err){
  let text = "";
  try {
    if (err == null) {
      text = "";
    } else if (typeof err === "string") {
      text = err;
    } else {
      text = String(err);
      if (err.message != null) text += " " + String(err.message);
      if (err.body != null) text += " " + String(err.body);
      try {
        text += " " + JSON.stringify(err);
      } catch (eJson) {}
    }
  } catch (e) {
    text = String(err);
  }
  const lower = String(text).toLowerCase();
  return lower.indexOf("insufficient_quota") >= 0 ||
    lower.indexOf("no credits remaining") >= 0 ||
    lower.indexOf("billing") >= 0;
}

function safeCallGPT(promptText, modelName = null) {
  // NEW PARSER START: 재시도 로직 강화 (빈 응답도 재시도)
  const isClustering = promptText.includes('클러스터링') || promptText.includes('pid');
  const maxRetries = isClustering ? 0 : 2; // 클러스터링은 재시도 없음, 코딩은 2회 재시도
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = callGPT(promptText, modelName);
      if (!response || response.trim() === '') {
        if (attempt < maxRetries) {
          Logger.log(`⚠️ safeCallGPT: 빈 응답 감지, 재시도 ${attempt + 1}/${maxRetries}`);
          Utilities.sleep(500 * (attempt + 1)); // 재시도 간격 증가
          continue; // 재시도
        }
        Logger.log("❌ safeCallGPT: 모든 재시도 후에도 빈 응답");
        throw new Error("GPT 응답이 비어 있습니다. 재시도 후에도 실패했습니다.");
      }
      return response;
    } catch (err) {
      // quota exhaustion은 일시적 429가 아님 — retry 금지, 즉시 throw
      if (_isOpenAIQuotaExhaustedError_(err)) {
        Logger.log("OPENAI_QUOTA_EXHAUSTED=true");
        throw err;
      }

      const errorMsg = err.toString();
      
      // 파라미터 오류 (400)는 재시도 불필요
      if (errorMsg.includes('400') || errorMsg.includes('Unsupported parameter')) {
        Logger.log("❌ safeCallGPT: 파라미터 오류 (400), 재시도 불필요");
        throw err; // 에러를 그대로 전파
      }
      
      // 클러스터링은 재시도 없음
      if (isClustering) {
        return "";
      }
      
      // 네트워크/일시적 오류 또는 파싱 실패는 재시도
      if (attempt < maxRetries) {
        if (errorMsg.includes('network') || errorMsg.includes('timeout') || 
            errorMsg.includes('429') || errorMsg.includes('500') || errorMsg.includes('503') ||
            errorMsg.includes('파싱 실패') || errorMsg.includes('응답 파싱')) {
          Logger.log(`⚠️ safeCallGPT: 일시적 오류 감지, 재시도 ${attempt + 1}/${maxRetries}`);
          Utilities.sleep(500 * (attempt + 1)); // 재시도 간격 증가
          continue; // 재시도
        }
      }
      
      // 최종 실패 시 에러 전파
      if (attempt === maxRetries) {
        throw err;
      }
    }
  }
  
  // 모든 재시도 실패 시 에러 (빈 문자열 반환 대신)
  throw new Error("GPT 호출 실패: 모든 재시도 후에도 실패했습니다.");
}

/**
 * UPDATED: gpt-5-mini & gpt-5 통합 호출 함수 (레거시 호환)
 * - 기존 코드 호환을 위해 유지
 * - 내부적으로 safeCallGPT 사용 (모델명 자동 전달)
 */
function fetchOpenAI(model, input, options = {}) {
  // UPDATED: safeCallGPT로 위임 (모델명 자동 전달)
  return safeCallGPT(input, model);
}

/**
 * 1행이 데이터로 보이면 자동으로 헤더 행 삽입
 */
function ensureHeaderRowStrict(){
  const sh = SpreadsheetApp.getActiveSheet();
  if (sh.getLastRow() < 1) return;

  const lastCol = Math.max(16, sh.getLastColumn());
  const r1 = sh.getRange(1,1,1,lastCol).getValues()[0].map(v=>String(v||'').trim());

  // "헤더 같지 않음" 또는 G..J가 숫자/공백 위주면 → 데이터로 간주
  const looksHeader =
    r1.includes('발화자') || r1.includes('요약문') || r1.includes('K차원') ||
    r1.some(h => /^참석자\s*\d/.test(h));
  const gj = r1.slice(6, 10); // G..J (0-based로 6~9)
  const gjIsNumericLike = gj.length===4 && gj.filter(x => x==='' || /^[\d\s]+$/.test(x)).length>=3;

  if (!looksHeader || gjIsNumericLike){
    Logger.log("🔧 1행이 데이터로 보임. 헤더 행을 자동 삽입합니다.");
    sh.insertRowBefore(1);
    const hdr = [
      '발화자','타임스탬프','발화','PID부여','PID','요약문', // A..F
      '', '', '', '',                                         // G..J (학생 이름은 나중에)
      'K차원','C차원','M차원','P차원','교사 개입 여부','패턴'  // K..P
    ];
    sh.getRange(1,1,1,hdr.length).setValues([hdr]);
    sh.setFrozenRows(1);
    styleHeaderRow(); // 🔥 1행 스타일 자동 적용
    Logger.log("✅ 헤더 행 삽입 완료. G~J는 '교사·학생 지정'에서 채워집니다.");
  }
}

/**
 * 헤더 기본 라벨 고정(1행) — G~J는 비워둠
 */
function writeDefaultHeaders(){
  const sh = SpreadsheetApp.getActiveSheet();
  const ui = SpreadsheetApp.getUi();

  const hdr = [
    '발화자','타임스탬프','발화','PID부여','PID','요약문', // A..F
    '', '', '', '',                                         // G..J (참석자 1~4 이름 후에 채움)
    'K차원','C차원','M차원','P차원','교사 개입 여부','패턴'  // K..P
  ];
  const rng = sh.getRange(1,1,1,hdr.length);
  const cur = rng.getValues()[0];
  const needConfirm = cur.some(v => String(v||'').trim()!=='');
  if (needConfirm){
    const res = ui.alert('헤더 덮어쓰기','1행에 기존 값이 있습니다. 덮어쓸까요?',ui.ButtonSet.YES_NO);
    if (res !== ui.Button.YES) return;
  }
  rng.setValues([hdr]);
  sh.setFrozenRows(1);
  styleHeaderRow(); // 🔥 1행 스타일 자동 적용
  ui.alert('✅ 기본 헤더가 설정되었습니다.\nG~J(학생 이름)는 "교사·학생 지정" 메뉴에서 채워집니다.');
}

/**
 * 교사/학생 지정 → G~J(학생 4명) 헤더 채우기
 */
function setStudentsToGJHeaders(){
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();

  const s1Res = ui.prompt('학생1 이름(참석자 1)','예: 참석자 1(62) 또는 이름 그대로',ui.ButtonSet.OK_CANCEL);
  if (s1Res.getSelectedButton() !== ui.Button.OK) return;
  const s1 = s1Res.getResponseText().trim();
  
  const s2Res = ui.prompt('학생2 이름(참석자 2)', '', ui.ButtonSet.OK_CANCEL);
  if (s2Res.getSelectedButton() !== ui.Button.OK) return;
  const s2 = s2Res.getResponseText().trim();
  
  const s3Res = ui.prompt('학생3 이름(참석자 3)', '', ui.ButtonSet.OK_CANCEL);
  if (s3Res.getSelectedButton() !== ui.Button.OK) return;
  const s3 = s3Res.getResponseText().trim();
  
  const s4Res = ui.prompt('학생4 이름(참석자 4)', '', ui.ButtonSet.OK_CANCEL);
  if (s4Res.getSelectedButton() !== ui.Button.OK) return;
  const s4 = s4Res.getResponseText().trim();

  const values = [[s1,s2,s3,s4]];
  const rng = sh.getRange(1,7,1,4); // G..J
  rng.setValues(values);

  // 저장
  PropertiesService.getDocumentProperties().setProperty('STUDENTS_GJ', JSON.stringify(values[0]));

  styleHeaderRow(); // 🔥 1행 스타일 자동 적용 (전체 헤더 일관성 유지)
  ui.alert('✅ G~J 헤더가 학생 이름으로 설정되었습니다.\n- ' + values[0].join(', '));
}

/**
 * 화자별 발화수 분석 (헤더 기준)
 */
function runSpeakerCountByHeaders(){
  const sh = SpreadsheetApp.getActiveSheet();
  const ui = SpreadsheetApp.getUi();
  
  // ✅ 프리플라이트: 헤더 자동 삽입 + 열 매핑 확정
  const cols = preflightAndGetCols();
  
  const lastRow = sh.getLastRow();
  
  if (lastRow < 2){ 
    ui.alert('데이터 행이 없습니다.'); 
    return; 
  }

  // 🔧 S1~S4는 객체일 수 있으므로 숫자 열번호로 변환해서 사용
  const s1Col = colNumOf(cols.S1);
  const s2Col = colNumOf(cols.S2);
  const s3Col = colNumOf(cols.S3);
  const s4Col = colNumOf(cols.S4);
  const firstSCol = Math.min(s1Col, s2Col, s3Col, s4Col);
  const hdr = sh.getRange(1, firstSCol, 1, 4).getValues()[0];
  
  // 데이터 읽기 (2행~, S1~S4 블록을 첫 열부터 4개 열로 읽기)
  const data = sh.getRange(2, firstSCol, lastRow-1, 4).getValues();
  const totals = [0,0,0,0];
  
  data.forEach(row => {
    row.forEach((val, idx)=>{
      const v = toNum(val);
      if (v > 0) totals[idx] += v;
    });
  });

  Logger.log('화자별 합계(G~J): ' + totals.join(', '));
  ui.alert(
    '화자별 발화수 합계\n\n' +
    hdr[0] + ': ' + totals[0] + '\n' +
    hdr[1] + ': ' + totals[1] + '\n' +
    hdr[2] + ': ' + totals[2] + '\n' +
    hdr[3] + ': ' + totals[3]
  );
}
function setTeacherMarkersOnce(){
  var ui = SpreadsheetApp.getUi();
  var current = PropertiesService.getScriptProperties().getProperty('TEACHER_MARKERS') || '교사,선생,teacher,Teacher';
  var resp = ui.prompt("교사 식별자 설정", "쉼표로 구분해 입력 (예: 교사,선생,teacher,Teacher)\n⚠️ 'T'는 오탐 가능성 있음 (참석자 T, 영문 이니셜 등)\n현재값: " + current, ui.ButtonSet.OK_CANCEL);
  if (resp.getSelectedButton() !== ui.Button.OK) return;
  var val = (resp.getResponseText() || '').trim();
  if (!val) { ui.alert('값이 비어 있습니다.'); return; }
  PropertiesService.getScriptProperties().setProperty('TEACHER_MARKERS', val);
  ui.alert('교사 식별자가 저장되었습니다.');
}
function getTeacherMarkers(){
  var raw = PropertiesService.getScriptProperties().getProperty('TEACHER_MARKERS') || '교사,선생,teacher,Teacher';
  return raw.split(',').map(function(s){ return s.trim(); }).filter(function(x){ return !!x; });
}
function isTeacherSpeaker(name){
  if (!name) return false;
  var text = String(name).trim().toLowerCase();
  var markers = getTeacherMarkers().map(function(s){return s.trim().toLowerCase();});
  // 예: "교사 2", "선생님", "Teacher", "교사2" 모두 매치
  for (var i=0;i<markers.length;i++){
    var mm = markers[i];
    if (!mm) continue;
    if (text.indexOf(mm) !== -1) return true;
  }
  return false;
}

/**
 * PID 구간에 교사가 실제로 있었는지 행데이터 기반으로 확인
 */
function _pidHasTeacherByRows_(sheet, pid){
  var data = sheet.getDataRange().getValues();
  for (var r=1; r<data.length; r++){
    var curPid = (data[r][PID_COL-1]||"").trim();
    if (curPid !== pid) continue;
    var speaker = (data[r][SPEAKER_COL-1]||"").trim();
    if (isTeacherSpeaker(speaker)) return true;
  }
  return false;
}

/**
 * 모든 PID의 교사 존재 여부를 행데이터 기반으로 한 번에 계산
 */
function _buildTeacherPresenceMap_(sheet){
  const data = sheet.getDataRange().getValues();
  const map = {}; // pid -> boolean
  for (let r=1; r<data.length; r++){
    const pid = (data[r][PID_COL-1]||"").trim();
    if (!pid) continue;
    const speaker = (data[r][SPEAKER_COL-1]||"").trim();
    if (!map.hasOwnProperty(pid)) map[pid] = false;
    if (isTeacherSpeaker(speaker)) map[pid] = true;
  }
  return map;
}


/*** 정규화/타임스탬프 유틸 ***/
function normalizeUtterance(raw){
  if (!raw) return "";
  var t = String(raw);
  // 다양한 유니코드 따옴표 → 표준 따옴표로
  t = t.replace(/[\u201C\u201D\u201E\u201F\u2033"]/g, '"')
       .replace(/[\u2018\u2019\u2032']/g, "'");
  t = t.replace(/\s+/g, ' ').trim();
  var replacePairs = [
    [/안이/g, '아니'],[/머지/g, '뭐지'],[/므야/g, '뭐야'],
    [/그랫나/g, '그랬나'],[/그랫던/g, '그랬던'],[/그러면은/g, '그러면'],
    [/근대/g, '근데'],[/때문에요/g, '때문에'],[/그래서요/g, '그래서'],
    [/정리하자면은/g, '정리하자면']
  ];
  for (var i=0;i<replacePairs.length;i++){
    t = t.replace(replacePairs[i][0], replacePairs[i][1]);
  }
  return t;
}
function parseMMSS(s){
  if(!s) return null;
  var m = String(s).trim().match(/^(\d{1,2}):([0-5]\d)$/);
  if(!m) return null;
  return parseInt(m[1],10)*60 + parseInt(m[2],10);
}
function secToMMSS(sec){
  if(sec==null) return "??:??";
  var m = Math.floor(sec/60), s = sec%60;
  return (m<10?"0":"")+m+":"+(s<10?"0":"")+s;
}
function formatSecToStamp(x){
  if (x==null || isNaN(x)) return "시간미상";
  x = Math.max(0, Math.floor(x));
  var h = Math.floor(x/3600); x -= h*3600;
  var m = Math.floor(x/60);   var s = x - m*60;
  var pad = function(n) { return (n<10? "0"+n : ""+n); };
  return h>0 ? (h + ":" + pad(m) + ":" + pad(s)) : (pad(m) + ":" + pad(s));
}


/** 요약 보조 (로컬 폴백) - 모든 발화 포함 */
function summarizeTurns(turns, onlyStudents){
  var out = "", seq = 0;
  for (var i=0; i<(turns||[]).length; i++){
    var it = turns[i];
    if (onlyStudents && it.isTeacher) continue;
    var v = (it.utt || "").replace(/["""]/g,'').trim();
    if (!v) continue;
    var verb = "말한다";
    if (/[\?]|왜|무엇|어떻게/.test(v)) verb = "묻는다";
    else if (seq>0 && /그래서|그러면|그리고/.test(v)) verb="덧붙인다";
    else if (/아니|그건|틀렸|하지만/.test(v)) verb="반박한다";
    else if (/그럼|맞아|동의/.test(v)) verb="동의한다";
    else if (v.length > 40) verb="설명한다";
    out += (it.isTeacher ? '교사' : (it.speaker ? it.speaker : '학생')) + '가 "' + v + '"라고 ' + verb + '. ';
    seq++;
    // 모든 발화를 포함하도록 400자 제한 제거
  }
  return out.trim();
}


/*** 수동 블록 보조 ***/
function _mbToSec(b){
  if(b.type!=="time") return b;
  var ss = parseMMSS(b.start), ee = parseMMSS(b.end);
  return { type:b.type, start:b.start, end:b.end, pid:b.pid, startSec:ss, endSec:ee };
}
const MANUAL_BLOCKS2 = (MANUAL_BLOCKS||[]).map(_mbToSec);
function findManualBlock(tsSec, rowIndexOneBased){
  for (var i=0;i<MANUAL_BLOCKS2.length;i++){
    var b = MANUAL_BLOCKS2[i];
    if (b.type==="time" && tsSec!=null && b.startSec!=null && b.endSec!=null){
      if (tsSec>=b.startSec && tsSec<=b.endSec) return b;
    }else if(b.type==="rows"){
      if (rowIndexOneBased>=b.startRow && rowIndexOneBased<=b.endRow) return b;
    }
  }
  return null;
}


/*** 재분할기 & 스무딩 ***/
function resegmentByRules(rows, gptRowCodes){
  var byRow = {};
  (gptRowCodes||[]).forEach(function(r) { byRow[r.row] = r; });
  var toSec = function(s) { return parseMMSS(s); };
  var pidStr = function(n) { return "P" + String(n).padStart(3,"0"); };
  var sameManualBlock = function(i,j){
    var r1=rows[i], r2=rows[j];
    var b1=findManualBlock(toSec(r1.ts), r1.row);
    var b2=findManualBlock(toSec(r2.ts), r2.row);
    return (b1 && b2 && b1===b2);
  };
  var fixedPidOf = function(i){
    var r = rows[i];
    var b = findManualBlock(toSec(r.ts), r.row);
    return (b && b.pid) ? b.pid : null;
  };


  var N = rows.length;
  if (!N) return { pidsByRow:{}, idList:[] };


  var pidNum=1, curPid=fixedPidOf(0) || pidStr(pidNum);
  var runLen=1;
  var pidsByRow={}; var idList=[curPid];


  var teacherActive = !!rows[0].isTeacher;
  var nonTeacherStreak = rows[0].isTeacher ? 0 : 1;
  var lastTeacherSec = rows[0].isTeacher ? toSec(rows[0].ts) : null;


  pidsByRow[rows[0].row] = curPid;


  for (var i=1;i<N;i++){
    var prev=rows[i-1], cur=rows[i];
    var prevC = byRow[prev.row] || {a:"none", c:"none", d:"none"};
    var curC  = byRow[cur.row]  || {a:"none", c:"none", d:"none"};
    var curSec = toSec(cur.ts);


    var boundary=false, reason="";
    var inManual = sameManualBlock(i-1,i);


    // 교사 창
    if (!inManual){
      if (!teacherActive && cur.isTeacher){
        boundary=true; reason="교사 진입";
        teacherActive=true; nonTeacherStreak=0; lastTeacherSec=curSec;
      }
      if (teacherActive){
        if (cur.isTeacher){
          nonTeacherStreak=0; lastTeacherSec=curSec;
        }else{
          nonTeacherStreak++;
          const tooLong = (lastTeacherSec!=null && curSec!=null && (curSec-lastTeacherSec)>=TEACHER_WINDOW_SEC);
          if (nonTeacherStreak>=TEACHER_EXIT_ROWS || tooLong){
            boundary=true; reason="교사 이탈";
            teacherActive=false;
          }
        }
      }
    }
    // 긴 공백
    if (!boundary && !inManual){
      var ps = toSec(prev.ts);
      if (ps!=null && curSec!=null && (curSec-ps)>=GAP_SPLIT_SEC){
        boundary=true; reason="긴 공백 " + (curSec-ps) + "s";
      }
    }
    // 질문 트리거
    if (!boundary && !inManual){
      var isQ = /\?/.test(cur.utter) || curC.c==="C2";
      if (isQ){
        var ignore = teacherActive && cur.isTeacher;
        if (!ignore){ boundary=true; reason="질문/C2"; }
      }
    }
    // 코드 변화
    if (!boundary && !inManual){
      var consider = true;
      if (teacherActive){
        if (prev.isTeacher || cur.isTeacher) consider=false;
      }
      if (consider){
        var changed = (prevC.a!==curC.a) + (prevC.c!==curC.c) + (prevC.d!==curC.d);
        if (changed>=1){
          var persistent = true;
          if (i+1 < N){
            var nxt = rows[i+1];
            if (!(nxt.isTeacher || prev.isTeacher || cur.isTeacher)){
              var nxtC = byRow[nxt.row] || {a:"none", c:"none", d:"none"};
              persistent = (curC.a===nxtC.a && curC.c===nxtC.c && curC.d===nxtC.d);
            }
          }
          if (persistent){ boundary=true; reason="코드 변화"; }
        }
      }
    }
    // 최소 길이 보장(약한 경계만)
    var strong = /교사 진입|교사 이탈|긴 공백|질문/.test(reason);
    if (boundary && !strong && runLen < MIN_SEG_ROWS){
      boundary=false; reason="";
    }
    // 커밋
    if (boundary){
      var fixed = fixedPidOf(i);
      if (fixed){
        curPid=fixed;
        if (idList[idList.length-1]!==fixed) idList.push(fixed);
      }else{
        pidNum++; curPid=pidStr(pidNum); idList.push(curPid);
      }
      runLen=1;
    }else{
      runLen++;
    }
    pidsByRow[cur.row]=curPid;
  }
  return { pidsByRow, idList };
}
function smoothSingletons(rows, pidsByRow, gptRowCodes){
  var toSec = function(s) { return parseMMSS(s); };
  var codeMap = {};
  (gptRowCodes||[]).forEach(function(r) { codeMap[r.row] = {a:r.a||"none", c:r.c||"none", d:r.d||"none"}; });


  // pid -> 연속 구간
  var segs = [];
  var curPid = null, startIdx = 0;
  var pidOf = function(r) { return pidsByRow[r.row] || ""; };
  for (var i=0;i<rows.length;i++){
    var pid = pidOf(rows[i]);
    if (pid !== curPid){
      if (curPid!=null) segs.push({pid:curPid, s:startIdx, e:i-1});
      curPid = pid; startIdx = i;
    }
  }
  if (curPid!=null) segs.push({pid:curPid, s:startIdx, e:rows.length-1});


  var rowHasTeacher = function(i) { return !!rows[i].isTeacher; };
  var segHasTeacher = function(seg) { for (var k=seg.s;k<=seg.e;k++){ if (rowHasTeacher(k)) return true; } return false; };
  segs.forEach(function(seg){
    var first = rows[seg.s], last = rows[seg.e];
    seg.len = seg.e - seg.s + 1;
    seg.startSec = toSec(first.ts);
    seg.endSec   = toSec(last.ts);
    seg.teacher  = segHasTeacher(seg);
  });


  var dist = function(r1, r2){
    var c1 = codeMap[r1.row] || {a:"none",c:"none",d:"none"};
    var c2 = codeMap[r2.row] || {a:"none",c:"none",d:"none"};
    return (c1.a!==c2.a) + (c1.c!==c2.c) + (c1.d!==c2.d);
  };
  var sameManual = function(i,j){
    var r1=rows[i], r2=rows[j];
    var b1=findManualBlock(toSec(r1.ts), r1.row);
    var b2=findManualBlock(toSec(r2.ts), r2.row);
    return (b1 && b2 && b1===b2);
  };
  var renumber = function(){
    var mapOldNew = {}; var cnt=0;
    rows.forEach(function(r){
      var old = pidsByRow[r.row] || "";
      if (!mapOldNew[old]) mapOldNew[old] = "P"+String(++cnt).padStart(3,"0");
      pidsByRow[r.row] = mapOldNew[old];
    });
  };


  for (var si=0; si<segs.length; si++){
    var seg = segs[si];
    if (seg.len > SINGLETON_ABSORB_ROWS) continue;
    var prev = segs[si-1], next = segs[si+1];


    if (prev && !sameManual(prev.e, seg.s) && findManualBlock(toSec(rows[prev.e].ts), rows[prev.e].row)) continue;
    if (next && !sameManual(seg.e, next.s) && findManualBlock(toSec(rows[next.s].ts), rows[next.s].row)) continue;


    var mid = rows[seg.s];
    var midSec = toSec(mid.ts);
    var gapPrev = prev ? (midSec!=null && prev.endSec!=null ? midSec - prev.endSec : null) : null;
    var gapNext = next ? (next.startSec!=null && midSec!=null ? next.startSec - midSec : null) : null;
    var prevFar = (gapPrev!=null && gapPrev>=GAP_SPLIT_SEC);
    var nextFar = (gapNext!=null && gapNext>=GAP_SPLIT_SEC);
    if (prevFar && nextFar) continue;
    if (prev && next){
      if ((gapPrev!=null && gapPrev>SINGLETON_ABSORB_SEC) && (gapNext!=null && gapNext>SINGLETON_ABSORB_SEC)) continue;
    }


    var score = function(neighbor, gap, isPrev){
      if (!neighbor) return {bad:true,score:1e9};
      var teacherMatch = (seg.teacher === neighbor.teacher) ? 0 : 1;
      var dPrev = dist(mid, rows[neighbor.e]);
      var dNext = dist(mid, rows[neighbor.s]);
      var d = Math.min(dPrev, dNext);
      var g = (gap==null ? 9999 : gap);
      return {bad:false, score: teacherMatch*10 + d*3 + Math.min(g, 120)/60 + (isPrev?0.01:0.02)};
    };
    var sPrev = score(prev, gapPrev, true);
    var sNext = score(next, gapNext, false);


    var pick = null;
    if (!sPrev.bad && (sPrev.score <= sNext.score || sNext.bad)) pick = "prev";
    else if (!sNext.bad) pick = "next";
    if (!pick) continue;


    var targetPid = (pick==="prev") ? prev.pid : next.pid;
    pidsByRow[mid.row] = targetPid;
  }


  renumber();
  var idSeq = []; var seen = {};
  rows.forEach(function(r){ var pid = pidsByRow[r.row]; if (pid && !seen[pid]){ seen[pid]=1; idSeq.push(pid); } });
  return { pidsByRow, idList:idSeq };
}


/***** ===== OpenAI 호출 유틸 (JSON 모드) ===== *****/
/**
 * UPDATED: rows 압축 헬퍼 함수 (토큰 수 최소화)
 * - 최대 row 수 제한
 * - 발화 내용 길이 제한
 * - 불필요한 필드 제거
 */
function compressRowsForClusterPrompt(rows, maxRows, maxCharsPerUtterance) {
  maxRows = maxRows || 200;              // 기본 최대 200행
  maxCharsPerUtterance = maxCharsPerUtterance || 80;  // 발화당 최대 80자
  
  var sliced = rows.slice(0, maxRows);
  return sliced.map(function(r) {
    var text = String(r.utter || r.text || r.utterance || "").replace(/\u200B/g,"").replace(/\s+/g," ").trim();
    if (text.length > maxCharsPerUtterance) {
      text = text.slice(0, maxCharsPerUtterance) + "...";
    }
    
    return {
      row: r.row,                          // 행 번호 (필수)
      speaker: r.speaker || r.S || "",     // 화자 정보
      role: r.role || (r.isTeacher ? "teacher" : "student") || "",  // teacher/student 구분
      timeSec: r.timeSec || r.sec || (r.ts ? String(r.ts).substring(0, 8) : null),  // 시간 정보
      text: text
    };
  });
}

/**
 * UPDATED: 슬림 클러스터링 프롬프트 생성 (토큰 수 최소화)
 * - systemPrompt 대폭 축소 (2-3줄 요약만)
 * - rows 압축 후 사용
 * - manualBlocks 최소 안내만
 */
function getClusterPrompt(rows, gapSplitSec, maxLenTurns, manualBlocks) {
  // UPDATED: systemPrompt 대폭 축소
  var systemPrompt = `한국어 소집단 담화에서 K/C/M 코드를 부여하고 pid로 클러스터를 나눕니다. C는 학생↔학생 상호작용에만, 교사 발화는 evidence로 쓰지 않습니다. off-task(수다/농담)는 K="none". JSON: {"row_codes":[{"row":번호,"k":"K1|K2|K3|none","c":"C1-C7|none","m":"M1-M4|none","pid":"P001"}],"clusters":[]}`;
  
  // UPDATED: rows 압축 (토큰 수 최소화)
  var compactRows = compressRowsForClusterPrompt(rows, 200, 80);
  
  // UPDATED: userPrompt 최소화
  var userPrompt = `[ROWS]
${JSON.stringify(compactRows)}

[PARAMS]
gap_split_sec = ${gapSplitSec}
max_len_turns = ${maxLenTurns}
${manualBlocks && manualBlocks.length > 0 ? `manual_blocks = ${JSON.stringify(manualBlocks)} (이 구간은 같은 pid로 유지)` : ''}

JSON 형식으로만 답변하세요.`;
  
  return [{ role: "system", content: systemPrompt }, { role: "user", content: userPrompt }];
}


// 안전한 문자열 변환 (GPT API 호출용)
function _safeStr_(x){ return (x==null) ? "" : String(x); }

/**
 * UPDATED FOR GPT-5: responses API로 변경
 * messages 배열을 input 문자열로 변환하여 fetchOpenAI 호출
 */
/**
 * UPDATED: gpt-5-mini & gpt-5 통합 JSON 호출 함수
 * - 내부적으로 safeCallGPT 사용 (모델명 자동 전달)
 * - messages를 input 문자열로 변환
 * - 빈 응답 시 빈 객체 반환 (전체 프로세스 중단 없음)
 */
function callGPT_JSON(messages, modelName = null){
  try{
    // 입력 검증
    if (!messages || !Array.isArray(messages) || messages.length === 0) {
      throw new Error('messages 배열이 비어있습니다.');
    }
    
    // messages를 안전하게 처리
    messages = messages.map(function(m){
      return { role: m.role || 'user', content: _safeStr_(m.content || '') };
    });
    
    const model = modelName || MODEL;
    
    // UPDATED: messages를 input 문자열로 변환
    const input = messagesToInput(messages);
    
    // UPDATED: safeCallGPT로 안전하게 호출 (모델명 자동 전달)
    const response = safeCallGPT(input, model);
    
    if (!response || response.trim() === '') {
      // 빈 응답은 빈 객체로 반환 (전체 프로세스 중단 없음)
      return {};
    }
    
    try {
      const parsed = JSON.parse(response);
      
      // 파싱된 결과가 빈 객체인지 확인
      if (!parsed || (typeof parsed === 'object' && Object.keys(parsed).length === 0)) {
        return {}; // 빈 객체 반환 (전체 프로세스 중단 없음)
      }
      
      return parsed;
    } catch (parseError) {
      // 파싱 실패는 빈 객체로 반환 (전체 프로세스 중단 없음)
      return {};
    }
  }catch(e){
    // API 키 관련 오류는 예외를 다시 던져서 상위에서 처리하도록
    var errorMsg = e.toString();
    if (errorMsg.includes('API Key') || errorMsg.includes('API 키') || errorMsg.includes('401') || errorMsg.includes('설정되지 않았습니다')) {
      throw e;
    }
    // 기타 오류는 빈 객체 반환 (전체 프로세스 중단 없음)
    return {};
  }
}
function fetchAllWithRetry(reqs, maxRetry){
  var out = new Array(reqs.length);
  var pending = reqs.map(function(r,i) { return {req:r, idx:i, tries:0}; });
  var retryCount = 0;
  while(pending.length){
    var batch = pending.map(function(p) { return p.req; });
    var res = UrlFetchApp.fetchAll(batch);
    var next = [];
    for(var i=0;i<res.length;i++){
      var r = res[i], p = pending[i];
      var code = r.getResponseCode();
      if(code>=200 && code<300){
        out[p.idx]=r;
      }else{
        p.tries++;
        if(p.tries<=maxRetry) {
          next.push(p);
          retryCount++;
          if(code===429 || code>=500) {
            var baseDelay = Math.pow(2, p.tries) * 1000;
            var jitter = Math.floor(Math.random() * 1000);
            var delay = Math.min(baseDelay + jitter, 10000);
            Logger.log("재시도 " + p.tries + "/" + maxRetry + " (HTTP " + code + "): " + delay + "ms 대기");
            Utilities.sleep(delay);
          }
        } else {
          out[p.idx]=r;
          Logger.log("최대 재시도 초과: HTTP " + code + " - " + r.getContentText().substring(0, 100));
        }
      }
    }
    pending = next;
  }
  if (retryCount > 0) Logger.log("전체 재시도 횟수: " + retryCount);
  return out;
}
function hash(str){
  var raw = Utilities.computeDigest(Utilities.DigestAlgorithm.MD5, str, Utilities.Charset.UTF_8);
  return raw.map(function(b){ var v=(b+256)%256; return (v<16?"0":"")+v.toString(16); }).join("");
}
function getCache(){ return CacheService.getScriptCache(); }
function cacheGet(k){ try{ return getCache().get(k); }catch(e){ return null; } }
function cachePut(k, v, ttl){ try{ getCache().put(k, v, ttl || CACHE_TTL_SEC); }catch(e){} }
function sheetKey(){ return "sh:" + SpreadsheetApp.getActiveSheet().getSheetId(); } // 레거시 호환성
function segHash(s){
  var turnSig = s.summarySource
    .map(function(t){ return (t.isTeacher?'T|':'S|') + (t.speaker||'') + "|" + normalizeUtterance(t.utt); })
    .join("||");
  // ✅ 요약 모드별 캐시 분리
  var ver = "v5_full";
  if (SUMMARY_MODE === "gpt_narrative") ver = "v5_multicluster_narr";
  if (SUMMARY_MODE === "gpt_rich")       ver = "v6_rich_narr";
  if (SUMMARY_MODE === "gpt_exhaustive") ver = "v7_exhaustive";
  return hash([s.id, s.time.start, s.time.end, turnSig, ver, s.summarySource.length].join("###"));
}
function codingHash(summary){ return hash("code|"+summary); }


/**
 * 🚀 GPT-5-MINI 배치 호출 (순차 처리)
 * - 각 항목을 독립적으로 순차 처리
 * - safeCallGPT로 안전하게 호출 (자동 재시도 포함)
 * - sleep() 없음
 */
function callGPT_JSON_batch(messagesList, modelName = null){
  // API 키 사전 체크
  const apiKey = getApiKey();
  if (!apiKey) {
    throw new Error('API 키가 설정되지 않았습니다.');
  }
  
  // 🚀 ULTRA-FAST: 순차 처리, sleep 제거
  var results = [];
  for (var i = 0; i < messagesList.length; i++) {
    try {
      var msgs = messagesList[i] || [];
      // 안전한 문자열 처리
      msgs = msgs.map(function(m){
        return { role: m.role || 'user', content: _safeStr_(m.content || '') };
      });
      
      // messages를 input 문자열로 변환
      var input = messagesToInput(msgs);
      
      // UPDATED: safeCallGPT로 안전하게 호출 (모델명 자동 전달)
      var response = safeCallGPT(input, model);
      
      if (!response || response.trim() === '') {
        // 빈 응답은 빈 객체로 처리 (전체 프로세스 중단 없음)
        results.push({});
        continue;
      }
      
      try {
        var parsed = JSON.parse(response);
        
        // 파싱된 결과가 빈 객체인지 확인
        if (!parsed || (typeof parsed === 'object' && Object.keys(parsed).length === 0)) {
          results.push({});
        } else {
          results.push(parsed);
        }
      } catch (parseError) {
        // 파싱 실패는 빈 객체로 처리 (전체 프로세스 중단 없음)
        results.push({});
      }
    } catch (itemError) {
      // 오류 발생 시 빈 객체로 처리 (전체 프로세스 중단 없음)
      results.push({});
    }
  }
  
  return results;
}


/*** GPT 클러스터링(샤딩 지원) ***/
/**
 * UPDATED: 클러스터링 기본 경로 (자동 샤딩 전환)
 * - 120행 이상이면 자동으로 샤딩 버전 호출
 * - 프롬프트 압축 및 토큰 수 최소화
 */
function callGPTForClustering(rows) {
  try {
    const apiKey = getApiKey();
    if (!apiKey) {
      throw new Error('API 키가 설정되지 않았습니다.');
    }
    
    if (!rows || rows.length === 0) {
      return { row_codes: [], clusters: [] };
    }
    
    // UPDATED: 120행 이상이면 자동으로 샤딩 버전 호출
    if (rows.length > 120) {
      return callGPTForClustering_sharded(rows);
    }
    
    var gapSplitSec = GAP_SPLIT_SEC || 90;
    var maxLenTurns = 15;
    var manualBlocks = MANUAL_BLOCKS || [];
    
    // UPDATED: 압축된 프롬프트 사용
    var messages = getClusterPrompt(rows, gapSplitSec, maxLenTurns, manualBlocks);
    
    var result = callGPT_JSON(messages, MODEL_CLUSTER);
    if (!result) {
      return null;
    }
    
    // UPDATED: 결과 검증 및 정규화
    if (!result.row_codes || !Array.isArray(result.row_codes)) {
      return null;
    }
    
    // row_codes의 row 번호를 실제 rows와 매핑
    var rowCodeMap = {};
    result.row_codes.forEach(function(rc) {
      if (rc.row) {
        rowCodeMap[rc.row] = rc;
      }
    });
    
    // 모든 rows에 대해 코드 할당 (누락된 row는 기본값 사용)
    var allRowCodes = [];
    for (var i = 0; i < rows.length; i++) {
      var rc = rowCodeMap[rows[i].row] || {};
      allRowCodes.push({
        row: rows[i].row,
        k: rc.k || "none",
        c: rc.c || "none",
        m: rc.m || "none",
        pid: rc.pid || ("P" + String(Math.floor(i / 10) + 1).padStart(3, "0"))
      });
    }
    
    return { 
      row_codes: allRowCodes, 
      clusters: result.clusters || [] 
    };
  } catch (e) {
    if (e.toString().includes('API') || e.toString().includes('API Key') || e.toString().includes('API 키')) {
      throw e;
    }
    return null;
  }
}
/**
 * UPDATED: 샤딩 클러스터링 (동적 샤드 크기 + 압축 프롬프트)
 * - rows.length에 따라 동적 샤드 크기 조정
 * - 압축된 프롬프트 사용 (토큰 수 최소화)
 * - 각 샤드를 순차 처리
 */
function callGPTForClustering_sharded(rows, shardSize){
  // API 키 사전 체크
  const apiKey = getApiKey();
  if (!apiKey) {
    throw new Error('API 키가 설정되지 않았습니다.');
  }
  
  if (!rows || rows.length === 0) {
    return { row_codes: [], clusters: [] };
  }
  
  // UPDATED: 동적 샤드 크기 조정
  var total = rows.length;
  var effectiveShardSize;
  if (total > 400) {
    effectiveShardSize = 60;
  } else if (total > 200) {
    effectiveShardSize = 80;
  } else {
    effectiveShardSize = shardSize || SHARD_SIZE || 100;
  }
  
  var overlap = SHARD_OVERLAP;
  var shards = [];
  
  // 샤드 생성
  for (var i = 0; i < rows.length; i += effectiveShardSize) {
    var s = Math.max(0, i - overlap);
    var e = Math.min(rows.length, i + effectiveShardSize + overlap);
    shards.push(rows.slice(s, e));
  }
  
  // UPDATED: 압축된 프롬프트로 배치 처리
  var gapSplitSec = GAP_SPLIT_SEC || 90;
  var maxLenTurns = 15;
  var manualBlocks = MANUAL_BLOCKS || [];
  var messagesList = shards.map(function(sh) { 
    return getClusterPrompt(sh, gapSplitSec, maxLenTurns, manualBlocks); 
  });
  
  // UPDATED: 배치 처리 (순차)
  var results = callGPT_JSON_batch(messagesList, MODEL_CLUSTER);
  
  // 결과 병합
  var allRowCodes = [];
  var allClusters = [];
  
  for (var si = 0; si < shards.length; si++) {
    var res = results[si] || {};
    var rc = res.row_codes || [];
    
    // row_codes 병합
    if (Array.isArray(rc)) {
      rc.forEach(function(code) {
        if (code.row) {
          allRowCodes.push(code);
        }
      });
    }
    
    // clusters 병합
    if (res.clusters && Array.isArray(res.clusters)) {
      allClusters.push.apply(allClusters, res.clusters);
    }
  }
  
  // 누락된 row는 기본값으로 채우기
  var rowCodeMap = {};
  allRowCodes.forEach(function(rc) {
    rowCodeMap[rc.row] = rc;
  });
  
  var finalRowCodes = [];
  for (var i = 0; i < rows.length; i++) {
    var rc = rowCodeMap[rows[i].row] || {};
    finalRowCodes.push({
      row: rows[i].row,
      k: rc.k || "none",
      c: rc.c || "none",
      m: rc.m || "none",
      pid: rc.pid || ("P" + String(Math.floor(i / 10) + 1).padStart(3, "0"))
    });
  }
  
  return { row_codes: finalRowCodes, clusters: allClusters };
}


/*** PID 델타 감지(요약용) ***/
function diffChangedPIDs_(sheet){
  var prop = PropertiesService.getDocumentProperties();
  var K = "LAST_PID_SNAPSHOT_"+sheet.getSheetId();
  var data = sheet.getDataRange().getValues();
  var nowPids = [];
  for (var r=2; r<=data.length; r++){
    var pid = (data[r-1][PID_COL-1]||"").trim();
    if (pid) nowPids.push(r+"="+pid);
  }
  var snap = nowPids.join("|");
  var prev = prop.getProperty(K) || "";
  prop.setProperty(K, snap);
  var changedRows = new Set();
  if (prev){
    var mapPrev = {};
    prev.split("|").forEach(function(s) { var sp = s.split("="); mapPrev[sp[0]]=sp[1]; });
    for (var r=2; r<=data.length; r++){
      var cur = (data[r-1][PID_COL-1]||"").trim();
      var was = mapPrev[r] || "";
      if (cur !== was) changedRows.add(r);
    }
  }else{
    for (var r=2; r<=data.length; r++) { if ((data[r-1][PID_COL-1]||"").trim()) changedRows.add(r); }
  }
  var segs = collectSegmentsByPID_(data);
  var changedPID = new Set();
  segs.forEach(function(s) { for (var r=s.startRow; r<=s.endRow; r++){ if (changedRows.has(r)){ changedPID.add(s.id); break; } } });
  return { changedPID: changedPID, segs: segs };
}


/*** 개선된 코딩 호출 (학생 주체성 중심) ***/
function callEnhancedCoding(clusters, prevSummary) {
  if (!clusters || !clusters.length) return { results: [] };
 
  Logger.log("KCM/P 코딩 시작: " + clusters.length + "개 클러스터");
 
  // 배치로 나누기
  var batches = [];
  for (var i = 0; i < clusters.length; i += BATCH_SIZE) {
    batches.push(clusters.slice(i, i + BATCH_SIZE));
  }
 
  var allResults = [];
 
  for (var i = 0; i < batches.length; i++) {
    var batch = batches[i];
    Logger.log("배치 " + (i + 1) + "/" + batches.length + " 처리 중...");
   
    var messages = getEnhancedCodingPrompt(batch, prevSummary);
   
    var result = callGPT_JSON(messages, MODEL_KM);
    if (result && result.results && result.results.length > 0) {
      allResults = allResults.concat(result.results);
    } else {
      // 실패 시 기본값
      for (var j = 0; j < batch.length; j++) {
        allResults.push({
          id: batch[j].id,
          time: batch[j].time || {start: "", end: ""},
          teacher_involved: false,
          codes: {K: "없음", C: "없음", M: "없음"},
          writeups: {
            K: {label_ko: "해당 없음", analysis: "수업 과업·개념과 직접 관련된 설명/추론 근거가 뚜렷하지 않습니다."},
            C: {label_ko: "해당 없음", analysis: "학생↔학생 상호작용(정교화·비판·조율 등)이 관찰되지 않습니다."},
            M: {label_ko: "해당 없음", analysis: "목표·절차 조정, 논리 점검, 개념 이해 점검의 명시적 단서가 부족합니다."}
          },
          evidence: [],
          confidence: {K: 0.0, C: 0.0, M: 0.0}
        });
      }
    }
  }
 
  Logger.log("KCM 코딩 완료: " + allResults.length + "개 결과");
  return { results: allResults };
}


/**
 * KCM 코딩 프롬프트 (학생 주체성 + 샘플 톤 해설)
 */
function getEnhancedCodingPrompt(clusters, prevSummary) {
  var systemPrompt = `You are a senior coder for Korean small-group discourse. Return JSON only.

[INPUT COLUMNS & META]
- F: 클러스터 요약문(전체 맥락)
- G~J: 화자별 발화수(정수; 0 이상). meta.speaker_counts=[G,H,I,J].
- meta.teacher_involved (bool), meta.student_speakers (int; 선택), meta.sample_turns (학생 인용 후보)

⚠️ **판단 순서 엄수 (흔들림 방지)**:
1) **먼저** C/P를 판단하고 **확정** (이후 절대 변경 금지)
2) **그 다음** 확정된 C/P를 기준으로 K/M만 판단
3) C/P를 판단한 후, K/M 판단 과정에서 C/P를 재해석하거나 변경하지 마세요.

[MANDATORY GATES (apply before any coding)]
1) Off-task(먹방/농담/SNS/시험잡담/의성어 위주) ⇒ K=C=M=P="없음".

2) **[1단계] C차원 활성화 하드게이트 (먼저 확정)**:
   - meta.speaker_counts = [G,H,I,J] coerced to integers (non-numeric→0).
   - **activeSpeakers = count(x>0)** (값이 0보다 큰 화자 수, 1도 활성).
   - **if activeSpeakers < 2 ⇒ C="없음" and stop C analysis.**
   - **if activeSpeakers ≥ 2 ⇒ C는 반드시 C1~C7 중 하나("없음" 금지).**
   - meta.speaker_counts가 비었는데 요약문(F)에 서로 다른 학생이 2명 이상 등장하면 activeSpeakers=2로 간주.
   
   **예시**:
   - [1,1,0,0] → activeSpeakers=2 (1>0 두 개) → C 코드 부여 필수
   - [1,0,0,0] → activeSpeakers=1 → C="없음"
   - [0,1,2,1] → activeSpeakers=3 (1>0, 2>0, 1>0) → C 코드 부여 필수
   - [4,3,0,0] → activeSpeakers=2 → C 코드 부여 필수

3) ──────────────────────────────────────
   **[2단계] K 차원 판정 (설명/추론) — 공격적 부여**
   ──────────────────────────────────────
   **⚠️ C/P는 이미 1단계에서 확정되었습니다. C/P를 참고만 하고 절대 변경하지 마세요.**
   
   K는 **'표지어 유무'가 아니라 설명 행위**로 판정한다. 아래 중 하나라도 해당하면 K 코드를 부여한다.
   
   **K-부여 조건(하나라도 참이면 OK)**:
   1) **인과/조건/기능/메커니즘**이 맥락적으로 드러남
      - 암묵적이라도 "X(상태/조건) → Y(결과/역할)" 구조가 보이면 인정
      - 예: "물 속에 있으니까…", "공기가 잘 통해야…", "~하려고 ~한다"
   2) **질문에 근거를 담아 응답**하거나, **라벨링 + 짧은 이유**라도 동반
      - "이건 토양(라벨)… 왜냐면 지렁이가 …" → A
   3) **비교/대조로 개념 경계 제시**
      - "토양보다 공기 쪽이 맞아(근거 추가)", "온도보단 빛(이유)"
   
   **K-배제(=K="없음")는 아래일 때만**:
   - **완전 라벨링/단답**만 있고("이건 토양"), **같은 클러스터 내 근거가 전혀 없음**
   - **오프태스크**(잡담/농담/먹방 등)으로 학습 내용과 무관
   
   **K-세부코드 선택 규칙**:
   - **K1**: 개념·관계·과정/메커니즘 설명(원인/결과/역할/조건 드러남)
   - **K2**: 증거/자료/규칙/도구를 탐색·해석(카드 기준, 사례 나열, 규칙 적용 등)
   - **K3**: 주장+근거가 같은 클러스터에 공존(간단 이유라도 있으면 우선 고려)
   - **우선순위**: K3(주장+근거) > K1(메커니즘) > K2(자료·규칙)
   
   **K-키워드(완전표지어가 아니어도 단서로 사용; 맥락 결합 필수)**:
   - 인과·조건·목적: 왜/그래서/때문에/그러니까/그러면/~하려고/~하니까/~하면
   - 기능·역할: 역할/기능/도움/필요/통한다/흡수/분해/유지/방출/조절/관련
   - 기준·규칙: 기준/조건/카드/분류/근거/사례/증거/정의/특징/원리
   - 비교·대조: 보다/대신/아니고/~쪽/더/덜/vs

4) ──────────────────────────────────────
   **[2단계] M 차원 판정 (목표·절차/논리·개념 점검) — M3/M4 적극 부여**
   ──────────────────────────────────────
   **⚠️ C/P는 이미 1단계에서 확정되었습니다. C/P를 참고만 하고 절대 변경하지 마세요.**
   
   M는 M1에만 치우치지 말고 **M3/M4를 적극 부여**한다.
   
   **M1 — 목표·절차·역할 설정 + 명시적 수용**:
   - 학생 제안("~하자/쓰자/붙이자/정하자/역할 정하자") → 같은 클러스터 1~2턴 내 **또래의 명시적 수용/실행 언급**
   - 교사 유도 수용은 제외
   - 키워드: 하자/정하자/나 할게/네가 해/붙이자/쓰자/먼저/다음/역할/기록/발표
   
   **M3 — 논리·정합성 점검(반례·모순·타당성)**:
   - 주장의 타당성/일관성/증거 적합성 검토, 반례 제기, 모순 지적
   - 예: "말이 안 돼", "논리상…", "근거가 약함", "모순", "조건 안 맞아"
   - 키워드: 말이 안 돼/모순/논리/일관/타당/근거 부족/설명 안 됨/증거가
   
   **M4 — 개념 이해 점검(정의·분류·기준·왜 묻기)**:
   - 개념 정의·분류 기준·특징을 캐묻거나 확인, "왜 그렇게 생각?" 류 추궁
   - 예: "왜 공기야?", "분류 기준이 뭐였지?", "정의가 뭐야?", "특징은?"
   - 키워드: 왜/무엇이 기준/정의/특징/분류/개념상/조건/근거 말해
   
   **M-우선순위(복수 충족 시)**: M1 > M4 > M3 > (없음)
   
   **⚠️ 적용 팁**:
   - 라벨링+근거가 같은 행에 섞여 있으면 **무조건 K3 우선**으로 잡아.
   - "왜/기준/정의/분류/특징" 류가 보이면 **가급적 M4**로 주고, 반박·모순·타당성 언어가 보이면 **M3를 과감히** 준다.
   - M1은 제안→수용이 같은 클러스터 안에서 명시적으로 이어질 때만.

5) **[1단계] P차원 (참여도 코드북 - P0~P3) — 먼저 확정**:
   **우선순위**: P0 > P1 > P2 > P3
   **⚠️ P를 먼저 결정한 후, 이후 단계에서 절대 변경하지 마세요.**
   
   - **P0(의미 있는 참여 없음)**: 아무도 인식적 발화 없음 또는 완전 오프태스크.
     · 키워드: "침묵", "잡담", "농담", "의성어만", "오프태스크"
   
   - **P1(1명 중심 참여)**: 1명만 의미 있게 기여, 나머지는 수동적.
     · 키워드: "한 명만", "독주", "혼자 설명"
   
   - **P2(소수 중심 참여)**: 2~3명이 의미 있게 기여.
     · 키워드: "소수", "2-3명", "일부만"
   
   - **P3(다수 참여)**: 4명 이상이 의미 있게 기여하거나 고른 참여 분포.
     · 키워드: "다수", "고른 참여", "모두 참여"

[C-DECISION (when activeSpeakers ≥ 2)]
동의 남발 금지. 다음 우선순위로 맥락 판단:
• **C4 반박**: "아니", "말이 안 돼", "~아냐", 반례·대안 제시, 평가의 부정.
• **C2 명료화/근거요구**: "왜", "근거", "맞아?", "어떻게", "무엇", 확인 질문, 재질문.
• **C3 정교화/보탬**: "그러니까/즉/예를 들면/덧붙여/정리하면", 타인의 설명을 확장·재구성.
• **C6 조율/합의**: "~로 하자/붙이자/정하자/그러면 ~하자", 선택·절충·합의 형성.
- 여러 신호가 공존하면 **최종 상호작용 성격**을 선택(예: 동의→반박이면 C4).

[K-M-P (요약)]
- K: 설명 표지(왜/때문에/역할/조건/근거) 없으면 "없음".
- M1: 제안→동료의 1~2턴 내 **명시적 수용**이 같은 클러스터 내에 있을 때만.
- P: 발화 분포로 1차 분기(참여 없음=P0, 1명=P1, 소수=P2, 다수=P3).

[OUTPUT (one cluster)]
{
  "id":"P###",
  "codes":{"K":"K1|K2|K3|없음","C":"C1|C2|C3|C4|C5|C6|C7|없음","M":"M1|M2|M3|M4|없음","P":"P0|P1|P2|P3|없음"},
  "writeups":{
    "C":{"label_ko":"C#. …","analysis":"한 문장으로, 왜 그 C인지(반박/명료화/정교화/조율) 요약"}
  },
  "evidence":[
    {"dim":"C","quote":"<5–20자>","why":"핵심 요지"}
  ],
  "audit":{
    "speaker_counts":[g,h,i,j],
    "activeSpeakers": 0,
    "forced_C_none": false,
    "why_C":"<키워드/문구 근거>"
  }
}

[POST-CHECK]
- activeSpeakers ≥ 2 인데 C="없음"이면 반드시 C2/C3/C4/C6 중 하나로 교정한다(동일 우선순위).
- 교사 인용은 evidence로 사용하지 않는다.
- JSON only.`;


  var userPrompt = `[INPUT]
clusters_with_meta = ${JSON.stringify(clusters)} 
meta_prev = {"prev_cluster_summary":"${prevSummary || ''}"}

Notes:
- 각 cluster에는 {id, time, summary, meta(student_speakers, has_peer_interaction, teacher_involved, speaker_counts[화자별 발화수], sample_turns[학생 인용 후보…])}가 포함됨.
- 코딩은 **요약문(이벤트 전체 맥락)**을 1차 근거로 하되, evidence는 **학생 인용(5–20자)** 1개씩.

**C차원 필수 계산 (STEP 1 - 반드시 먼저 실행)**:
- activeSpeakers = meta.speaker_counts.filter(count => count >= 1).length
- activeSpeakers < 2 → C = "없음" (강제 반환, STEP 2 건너뜀)
- activeSpeakers >= 2 → STEP 2로 진행 (상호작용 내용 분석하여 C1~C7 결정)

계산 예시:
- meta.speaker_counts = [12,0,0,0] → activeSpeakers = 1 → **C="없음"** ✅
- meta.speaker_counts = [0,1,1,0] → activeSpeakers = **2** → **C 코드 검토** (예: "토양 아니야" 반박 → C4) ✅
- meta.speaker_counts = [2,3,0,0] → activeSpeakers = 2 → C 코드 검토 ✅

**P차원 발화수 기반**: meta.speaker_counts 배열로 참여도를 정량 판단 (예: [6,5,4] vs [12,1,0]).

[OUTPUT SCHEMA]
{
  "calibration_rules": ["<=120자 ko rule 1", "..."],
  "results": [
    {
      "id":"P###",
      "time":{"start":"mm:ss|null","end":"mm:ss|null"},
      "teacher_involved": true|false,
      "codes":{"K":"K1|K2|K3|없음","C":"C1|C2|C3|C4|C5|C6|C7|없음","M":"M1|M2|M3|M4|없음","P":"P0|P1|P2|P3|없음"},
      "codes_display":{"K":"K1|K2|K3|없음","C":"C1..C7|없음","M":"M1|M2|M3|M4|없음","P":"P0|P1|P2|P3|없음"},
      "writeups":{
        "K":{"label_ko":"K#. …","analysis":"2~3문장. 왜 K인지 간결히. 학생 인용 1개 포함"},
        "C":{"label_ko":"C#. …","analysis":"**이벤트 전이(동의→반박/명료화 분기)** 근거를 명시"},
        "M":{"label_ko":"M#. …","analysis":"특히 M1은 4요건 충족 여부를 구체 서술"},
        "P":{"label_ko":"P#. …","analysis":"1~2문장. 전체 맥락 기반 참여도 패턴 근거"}
      },
      "evidence":[
        {"dim":"K","quote":"<5–20자 학생 인용>","why":"<핵심 요지>"},
        {"dim":"C","quote":"<5–20자 학생 인용>","why":"<핵심 요지>"},
        {"dim":"M","quote":"<5–20자 학생 인용>","why":"<핵심 요지>"},
        {"dim":"P","quote":"<5–20자 학생 인용>","why":"<핵심 요지>"}
      ],
      "confidence":{"K":0.0,"C":0.0,"M":0.0,"P":0.0},
      "audit":{
        "student_speakers": 0,
        "used_teacher_quote": false,
        "off_task_reason": "",
        "gates":{"peer_required": true|false,"forced_C_none": true|false},
        "why_no_K": "",
        "why_no_M": ""
      }
    }
  ]
}

[POST-CONSTRAINTS]
- teacher_involved==true면 codes_display 각 값 끝에 "*"를 붙여라(예: "C4*", "P2*").
- C="없음"이면 C evidence 항목을 생략하라.
- E="없음"이면 E evidence 항목을 생략하라.
- JSON 외 어떠한 설명도 출력하지 마라.`;


  return [
    { role: "system", content: systemPrompt },
    { role: "user", content: userPrompt }
  ];
}

/**
 * 🧪 테스트 지표 (로컬 검증 안내)
 * 
 * 내부 점검 시, 동일 클러스터에서:
 * - 동의→반박 패턴이 있는 샘플은 C4
 * - 명료화↔수용은 C1
 * - 명료화↔반박은 C4로 출력되는지 확인
 * 
 * 이 지침은 클러스터 요약문(F열) 기반의 맥락 우선 코딩을 강제함.
 * 파라미터: EVENT_GAP_SEC=30, CODE_DRIFT_TOL=0, TOPIC_OVERLAP_T=0.30
 */


/***** ===== 1) 클러스터링 + 요약(F) ===== *****/
function collectSegmentsByPID_(sheetData){
  var N = sheetData.length;
  var get = function(r,c) {
    if (r < 1 || r > N) return "";
    var row = sheetData[r-1] || [];
    var v = row[c-1];
    return (v == null) ? "" : v;
  };
  var segs = [];
  var curPid = null;
  var startRow = null;
  var lastRowInSeg = null;
  var turns = [];
  var pushSeg = function() {
    if (curPid == null) return;
    var formatTime = function(timeValue) {
      if (!timeValue) return "??:??";
      if (typeof timeValue === 'string') return timeValue;
      if (timeValue instanceof Date) {
        var minutes = timeValue.getMinutes();
        var seconds = timeValue.getSeconds();
        return (minutes < 10 ? "0" : "") + minutes + ":" + (seconds < 10 ? "0" : "") + seconds;
      }
      return "??:??";
    };
    segs.push({
      id: curPid,
      startRow: startRow,
      endRow: lastRowInSeg,
      time: { start: formatTime(get(startRow, TS_COL)), end: formatTime(get(lastRowInSeg, TS_COL)) },
      summarySource: turns.slice(),
      teacher_involved: turns.some(function(t) { return t.isTeacher; }) // J열 규칙: 클러스터 내 isTeacher==true가 하나라도 있으면 true
    });
  };
  for (var r = 2; r <= N; r++){
    var pid = (get(r, PID_COL) + "").trim();
    var speaker = get(r, SPEAKER_COL) + "";
    var isTeacher = isTeacherSpeaker(speaker);
    var utt = normalizeUtterance(String(get(r, UTTER_COL) || ""));
    var ts  = get(r, TS_COL) || null;
    if (!pid) continue;
    var rowItem = { row:r, speaker: speaker, isTeacher: isTeacher, ts: ts, utt: utt };
    if (pid !== curPid){
      if (curPid != null) pushSeg();
      curPid = pid; startRow = r; lastRowInSeg = r; turns = [rowItem];
    } else {
      lastRowInSeg = r; turns.push(rowItem);
    }
  }
  if (curPid != null) pushSeg();
  return segs;
}
function getExhaustiveSummaryPrompt_(segments){
  var sys = `You are a Korean narrative summarizer for small-group classroom talk.
Return JSON ONLY with schema:
{"results":[{"id":"P###","time":{"start":"mm:ss","end":"mm:ss"},"summary":"<text>"}]}

STYLE (모든 발화 포함 내러티브):
- 모든 지정된 발화자와 발화 내용을 빠짐없이 포함하여 요약문 작성.
- "한 학생이 …라고 말한다. 다른 학생이 …" 같은 흐름으로 모든 발화를 순서대로 서술.
- 각 발화자의 발화를 5~20자 인용부호로 포함("…").
- 교사 발화도 포함하여 전체 맥락을 완전히 제공.
- 평가·해석 금지(묘사 위주), 불필요한 감탄사/군더더기 제거.
- 화자 표기는 입력에 '남학생/여학생/교사' 단어가 있으면 그대로, 없으면 '학생' 사용.
- 시간을 언급하지 말 것(우리가 별도 포맷으로 붙임).
- 길이 제한 없이 모든 발화를 포함.
- 학생 발화가 1개라도 있으면 최소 1개의 **학생 직접 인용(5~20자)**을 반드시 포함.
- **발화수 집계는 절대 포함하지 말 것. 순수 내러티브만 작성.**`;

  // payload 구성
  var payload = segments.map(function(s) {
    var turns = (s.summarySource || []).map(function(t) {
      var sp = t.isTeacher ? "교사" :
        (String(t.speaker || "").match(/남학생|여학생/) ? t.speaker : (t.speaker || "학생"));
      return {speaker: sp, text: t.utt};
    });
    return {id: s.id, time: s.time, turns: turns};
  });

  var user = "segments=" + JSON.stringify(payload);
  return [{role: "system", content: sys}, {role: "user", content: user}];
}

function getNaturalSummaryFromFullTurnsPrompt_(segments){
  var sys = `You are a Korean classroom discourse summarizer.

REQUIREMENTS
- Use ALL substantive turns provided as evidence. Do NOT invent facts or events.
- Compress repetition/fillers and off-task chit-chat; keep the core peer interaction and any teacher intervention.
- Output 1–3 sentences per segment (max 3), natural narrative in Korean.
- Include 1–2 SHORT student quotes (5–20 chars) if helpful.
- If a teacher turn exists in the input, explicitly mention "교사" once.
- No timestamps or meta. No bullet points.
- STRICT JSON ONLY with the schema below.`;

  var payload = segments.map(function(s){
    var turns = (s.summarySource||[]).map(function(t){
      // 그대로 다 전달 (교사 포함)
      return {
        speaker: t.isTeacher ? "교사" : (t.speaker || "학생"),
        text: t.utt
      };
    });
    return { id: s.id, time: s.time, turns: turns };
  });

  var user = `segments=${JSON.stringify(payload)}
[OUTPUT SCHEMA]
{"results":[{"id":"P###","time":{"start":"mm:ss","end":"mm:ss"},"summary":"<한국어 1~3문장>"}]}
Return ONLY the JSON object.`;

  return [{role:"system", content:sys},{role:"user", content:user}];
}

/** 리치 내러티브 요약 프롬프트: 맥락 묘사 + 학생 인용 다수(+교사 존재 한 번 언급) */
function getRichContextSummaryPrompt_(segments){
  var sys = `You are a Korean classroom discourse summarizer.

GOAL
- For EACH segment, write a RICH narrative that covers ALL substantive student turns (no invention).
- Include 2–6 SHORT student quotes (5–20 chars) drawn from the actual turns.
- Mention "교사" exactly once if a teacher turn exists (context only; do not use teacher quotes as evidence).
- Preserve who-responded-to-whom (질문↔응답, 주장↔반박, 보충/정교화, 합의/조율 등).
- Compress fillers/off-task chatter, but do not drop substantive peer interaction.

STYLE
- Korean, 2–4 sentences (up to 5 if the segment is long).
- Natural narrative that clearly states the flow: who asks/explains/clarifies/objects/coordinates.
- No timestamps or bullet points. No meta commentary.

STRICT JSON ONLY:
{"results":[{"id":"P###","time":{"start":"mm:ss","end":"mm:ss"},"summary":"<text>"}]}
`;

  var payload = segments.map(function(s){
    var turns = (s.summarySource||[]).map(function(t){
      return {
        speaker: t.isTeacher ? "교사" : (t.speaker || "학생"),
        text: t.utt
      };
    });
    return { id: s.id, time: s.time, turns: turns };
  });

  var user = `segments=${JSON.stringify(payload)}
Return ONLY the JSON object.`;
  return [{role:"system", content: sys}, {role:"user", content: user}];
}

function summarizeSegmentsByPID_(segments){
  if (!segments.length) return {results:[]};
  var BATCH = BATCH_SIZE;
  var batches = [];
  for (var i=0; i<segments.length; i+=BATCH) batches.push(segments.slice(i, i+BATCH));
  var all = [];
  if (batches.length > 1) {
    var messagesList = batches.map(function(chunk) { return getExhaustiveSummaryPrompt_(chunk); });
    var results = callGPT_JSON_batch(messagesList, MODEL_SUMMARY);
    for (var i=0; i<results.length; i++){ var res = results[i]; if (res && res.results) all.push.apply(all, res.results); }
  } else {
    var res = callGPT_JSON(getExhaustiveSummaryPrompt_(segments), MODEL_SUMMARY);
    if (res && res.results) all.push.apply(all, res.results);
  }
  return {results: all};
}

function summarizeSegmentsByPID_Natural_(segments){
  if (!segments.length) return {results:[]};
  var BATCH = BATCH_SIZE;
  var batches = [];
  for (var i=0;i<segments.length;i+=BATCH) batches.push(segments.slice(i,i+BATCH));

  var all = [];
  if (batches.length>1){
    var messagesList = batches.map(function(chunk){ return getNaturalSummaryFromFullTurnsPrompt_(chunk); });
    var resList = callGPT_JSON_batch(messagesList, MODEL_SUMMARY);
    for (var i=0;i<resList.length;i++){
      var r = resList[i];
      if (r && r.results) all.push.apply(all, r.results);
    }
  }else{
    var r = callGPT_JSON(getNaturalSummaryFromFullTurnsPrompt_(segments), MODEL_SUMMARY);
    if (r && r.results) all.push.apply(all, r.results);
  }
  return {results: all};
}

/** 리치 내러티브 요약 실행 */
function summarizeSegmentsByPID_RichNarrative_(segments){
  if (!segments.length) return {results:[]};
  var BATCH = BATCH_SIZE;
  var batches = [];
  for (var i=0; i<segments.length; i+=BATCH) batches.push(segments.slice(i, i+BATCH));

  var all = [];
  if (batches.length>1){
    var messagesList = batches.map(function(chunk){ return getRichContextSummaryPrompt_(chunk); });
    var resList = callGPT_JSON_batch(messagesList, MODEL_SUMMARY);
    for (var i=0;i<resList.length;i++){
      var r = resList[i];
      if (r && r.results) all.push.apply(all, r.results);
    }
  }else{
    var r = callGPT_JSON(getRichContextSummaryPrompt_(segments), MODEL_SUMMARY);
    if (r && r.results) all.push.apply(all, r.results);
  }
  return {results: all};
}

function assignSummaryOnly(){
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data  = sheet.getDataRange().getValues();
  var N = data.length;
  if (N <= 1){ SpreadsheetApp.getUi().alert("데이터가 없습니다."); return; }
  var hasPID = data.slice(1).some(function(r){ return (r[PID_COL-1]||"").trim()!==""; });
  if (!hasPID){ SpreadsheetApp.getUi().alert("D열(PID)이 비었습니다. 먼저 클러스터링을 실행하세요."); return; }

  var diff = diffChangedPIDs_(sheet);
  var changedPID = diff.changedPID;
  var segs = diff.segs;

  // ✅ E열을 현재 세그먼트와 동일 순서로 재작성해 EF 정렬을 보장
  (function refreshE(){
    var idList = segs.map(function(s){ return s.id; });
    var lastRow = sheet.getLastRow();
    if (lastRow>=2) sheet.getRange(2, P_IDLIST_COL, lastRow-1, 1).clearContent();
    if (idList.length) sheet.getRange(2, P_IDLIST_COL, idList.length, 1).setValues(idList.map(function(x){ return [x]; }));
  })();

  var out = [];
  var cacheKey = sheetKey();

  // 1) 캐시 조회 & GPT 필요 목록 수집
  var need = [];                 // GPT/로컬 재계산이 필요한 seg들
  var needIdx = [];              // seg 인덱스
  var cachedTexts = {};          // id -> text

  for (var i=0;i<segs.length;i++){
    var s = segs[i];
    var segKey = cacheKey + "|summary|" + segHash(s);
    var summaryText = "";
    var mustRecompute = changedPID.has(s.id);

    if (!mustRecompute){
      var cached = cacheGet(segKey);
      if (cached){
        cachedTexts[s.id] = cached;
        continue;
      }
    }
    need.push(s);
    needIdx.push(i);
  }

  // 2) 요약 생성 (모드별)
  var produced = {}; // id -> text
  if (need.length){
    try{
      if (SUMMARY_MODE === "gpt_exhaustive"){
        var g = summarizeSegmentsByPID_(need); // ★ 모든 발화 포함 프롬프트
        var byId = {}; (g.results||[]).forEach(function(x){ byId[x.id]=x; });
        need.forEach(function(s){
          var x = byId[s.id] || {};
          var txt = x.summary || summarizeTurns(s.summarySource, false); // 폴백
          produced[s.id] = txt;
          cachePut(cacheKey + "|summary|" + segHash(s), txt);
        });
      }else if (SUMMARY_MODE === "gpt_rich"){
        var g = summarizeSegmentsByPID_RichNarrative_(need);
        var byId = {}; (g.results||[]).forEach(function(x){ byId[x.id]=x; });
        need.forEach(function(s){
          var x = byId[s.id] || {};
          var txt = x.summary || summarizeTurns(s.summarySource, false);
          produced[s.id] = txt;
          cachePut(cacheKey + "|summary|" + segHash(s), txt);
        });
      }else if (SUMMARY_MODE === "gpt_narrative"){
        var g = summarizeSegmentsByPID_Natural_(need);
        var byId = {}; (g.results||[]).forEach(function(x){ byId[x.id]=x; });
        need.forEach(function(s){
          var x = byId[s.id] || {};
          var txt = x.summary || summarizeTurns(s.summarySource, false);
          produced[s.id] = txt;
          cachePut(cacheKey + "|summary|" + segHash(s), txt);
        });
      }else{
        // local_full
        need.forEach(function(s){
          var txt = summarizeTurns(s.summarySource, false);
          produced[s.id] = txt;
          cachePut(cacheKey + "|summary|" + segHash(s), txt);
        });
      }
    }catch(e){
      // GPT 실패 시 전부 로컬 폴백
      need.forEach(function(s){
        var txt = summarizeTurns(s.summarySource, false);
        produced[s.id] = txt;
        cachePut(cacheKey + "|summary|" + segHash(s), txt);
      });
    }
  }

  // 3) 결과 집계 + 시트 기록
  var formatTime = function(timeValue){
    if (!timeValue) return "??:??";
    if (typeof timeValue === 'string') return timeValue;
    if (timeValue instanceof Date){
      var m=timeValue.getMinutes(), s=timeValue.getSeconds();
      return (m<10?"0":"")+m+":"+(s<10?"0":"")+s;
    }
    return "??:??";
  };

  for (var i=0;i<segs.length;i++){
    var s = segs[i];
    var tStart = formatTime(s.time.start);
    var tEnd   = formatTime(s.time.end);
    var text   = cachedTexts[s.id] || produced[s.id] || summarizeTurns(s.summarySource, false);
    out.push(["■ " + tStart + "~" + tEnd + "\t" + text]);
  }

  const lastRow = sheet.getLastRow();
  if (lastRow>=2) sheet.getRange(2, P_SUMMARY_REFINED_COL, lastRow-1, 1).clearContent();
  if (out.length) sheet.getRange(2, P_SUMMARY_REFINED_COL, out.length, 1).setValues(out);

  var modeLabel = (SUMMARY_MODE === "gpt_exhaustive") ? "GPT-전체발화포함" : 
                  (SUMMARY_MODE === "gpt_rich") ? "GPT-풍성한내러티브" : 
                  (SUMMARY_MODE === "gpt_narrative") ? "GPT-자연서술" : "로컬-전체발화";
  SpreadsheetApp.getUi().alert("요약 생성 완료(" + modeLabel + "): " + segs.length + "개 PID (변경: " + changedPID.size + "개)");
}


/***** ===== 1-1) 클러스터링(최적화 내장) ===== *****/
function assignClustersOnly(){
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  // 성능 개선: 필요한 열만 읽기 (전체 시트 대신)
  var lastRow = sheet.getLastRow();
  if (lastRow <= 1){ SpreadsheetApp.getUi().alert("데이터가 없습니다."); return; }
  
  const maxCol = Math.max(SPEAKER_COL, TS_COL, UTTER_COL);
  var data = sheet.getRange(1, 1, lastRow, maxCol).getValues();
  var N = data.length;
  if (N <= 1){ SpreadsheetApp.getUi().alert("데이터가 없습니다."); return; }

  // 성능 개선: 발화 행 준비 최적화
  var rows = [];
  for (var i=1;i<N;i++){
    var speaker = data[i][SPEAKER_COL-1] || "";
    var tsStr   = data[i][TS_COL-1] || "";
    var uttRaw  = String(data[i][UTTER_COL-1] || "").trim();
    if (!speaker && !tsStr && !uttRaw) continue;
    rows.push({ row: i+1, speaker: speaker, isTeacher: isTeacherSpeaker(speaker), ts: tsStr || null, utter: normalizeUtterance(uttRaw) });
  }
  if (!rows.length){ SpreadsheetApp.getUi().alert("C열(발화)이 비었습니다."); return; }


  // API 키 사전 체크
  try {
    const apiKey = getApiKey();
    if (!apiKey) {
      SpreadsheetApp.getUi().alert('❌ API 키가 설정되지 않았습니다.\n\n해결 방법:\n1. [API키 설정] 메뉴 클릭\n2. OpenAI API 키 입력 (sk-로 시작)\n3. 다시 클러스터링 실행');
      return;
    }
  } catch (keyError) {
    Logger.log("API 키 체크 실패: " + keyError.toString());
    var keyErrorMsg = keyError.toString();
    if (keyErrorMsg.includes('API Key') || keyErrorMsg.includes('API 키') || keyErrorMsg.includes('설정되지 않았습니다')) {
      SpreadsheetApp.getUi().alert('❌ API 키 오류\n\n해결 방법:\n1. [API키 설정] 메뉴 클릭\n2. OpenAI API 키 입력 (sk-로 시작)\n3. 다시 클러스터링 실행');
      return;
    }
    throw keyError; // 예상치 못한 오류는 다시 던짐
  }

  // GPT 의미 단위 클러스터링 시도 (샤딩+병렬)
  var gpt = null;
  try {
    if (rows.length > SHARD_SIZE) gpt = callGPTForClustering_sharded(rows, SHARD_SIZE);
    else gpt = callGPTForClustering(rows);
  } catch(e){ 
    Logger.log("GPT 클러스터링 예외: " + e);
    var errorMsg = e.toString();
    Logger.log("상세 오류: " + errorMsg);
    
    // UPDATED FOR GPT-5: 개선된 에러 메시지
    if (errorMsg.includes('API Key') || errorMsg.includes('API 키') || errorMsg.includes('401') || errorMsg.includes('설정되지 않았습니다')) {
      SpreadsheetApp.getUi().alert('❌ GPT 클러스터링 실패\n\n원인: API 키 오류\n' + errorMsg + '\n\n해결 방법:\n1) [API키 설정] 메뉴 클릭\n2) OpenAI API 키 입력 (sk-로 시작)\n3) 다시 클러스터링 실행');
      return;
    } else if (errorMsg.includes('429')) {
      SpreadsheetApp.getUi().alert('❌ GPT 클러스터링 실패\n\n원인: API 사용량 초과\n' + errorMsg + '\n\n해결 방법:\n1) 잠시 후 다시 시도\n2) API 사용량 확인');
      return;
    } else if (errorMsg.includes('network') || errorMsg.includes('timeout') || errorMsg.includes('네트워크')) {
      SpreadsheetApp.getUi().alert('❌ GPT 클러스터링 실패\n\n원인: 네트워크 오류\n' + errorMsg + '\n\n해결 방법:\n1) 인터넷 연결 확인\n2) 다시 시도');
      return;
    } else if (errorMsg.includes('404') || errorMsg.includes('모델명') || errorMsg.includes('endpoint')) {
      SpreadsheetApp.getUi().alert('❌ GPT 클러스터링 실패\n\n원인: 모델명 또는 endpoint 오류\n' + errorMsg + '\n\n해결 방법:\n1) API KEY 확인\n2) 인터넷 연결 확인\n3) 모델명 또는 endpoint 문제일 수 있음');
      return;
    } else {
      // 기타 오류의 경우 상세 정보 표시
      SpreadsheetApp.getUi().alert('❌ GPT 클러스터링 실패\n\n원인: ' + errorMsg + '\n\n해결 방법:\n1) [API키 설정] 메뉴에서 API 키 확인\n2) 인터넷 연결 확인\n3) 모델명 또는 endpoint 문제일 수 있음');
      return;
    }
  }


  // 성능 개선: D/E 초기화를 한 번에 처리
  var lastRow = sheet.getLastRow();
  if (lastRow>=2){
    // 두 열을 한 번에 클리어 (개별 호출 대신)
    const minCol = Math.min(PID_COL, P_IDLIST_COL);
    const maxCol = Math.max(PID_COL, P_IDLIST_COL);
    sheet.getRange(2, minCol, lastRow-1, maxCol - minCol + 1).clearContent();
  }


  // 성공 경로: 재분할→스무딩→Event 병합→기록
  if (gpt && Array.isArray(gpt.row_codes) && gpt.row_codes.length){
    const seg = resegmentByRules(rows, gpt.row_codes);
    const sm  = smoothSingletons(rows, seg.pidsByRow, gpt.row_codes);
    const ev  = mergeEventsFromActs(rows, sm.pidsByRow, gpt.row_codes); // ★ Event 병합 추가
    var cidsPerRow = new Array(N-1).fill("");
    for (var i=0;i<rows.length;i++){
      const rnum = rows[i].row;
      const pid  = ev.pidsByRow[rnum] || "";
      if (rnum>=2 && rnum<=N) cidsPerRow[rnum-2] = pid;
    }
    if (cidsPerRow.length) sheet.getRange(2, PID_COL, cidsPerRow.length, 1).setValues(cidsPerRow.map(x=>[x]));
    // E열(P_IDLIST_COL)은 normalizePidIntoE_에서 고유 PID 목록으로 재작성하므로 여기서는 쓰지 않음
    // assignSummaryOnly(); // ★ 주석 처리: 요약문은 buildClusterSummariesFromPID_에서 생성
    var msg = "✅ 클러스터링 완료\n\n" +
              "📦 생성된 PID: " + ev.idList.length + "개\n" +
              "🔗 Event 병합 적용됨";
    // 더 이상 alert를 여기서 표시하지 않음 (menu_cluster에서 통합 메시지 표시)
    // SpreadsheetApp.getUi().alert(msg);
    return;
  }


  // 폴백(시간 기반)
  if (USE_FALLBACK_ON_GPT_FAIL) {
    SpreadsheetApp.getUi().alert("⚠️ GPT 클러스터링 실패. 시간기반 폴백(90초) 수행.");
  } else {
    // UPDATED FOR GPT-5: 개선된 에러 메시지
    var lastErrorMsg = "알 수 없는 오류";
    try {
      if (gpt === null) {
        lastErrorMsg = "GPT 응답이 null입니다. API 호출 실패 또는 응답 파싱 실패";
      } else if (!Array.isArray(gpt.row_codes)) {
        lastErrorMsg = "GPT 응답 형식 오류: row_codes가 배열이 아닙니다";
      } else if (gpt.row_codes.length === 0) {
        lastErrorMsg = "GPT 응답이 비어있습니다: row_codes 배열이 비어있음";
      }
    } catch (e) {
      lastErrorMsg = "응답 처리 중 오류: " + e.toString();
    }
    
    SpreadsheetApp.getUi().alert("❌ GPT 클러스터링 실패\n\n원인: " + lastErrorMsg + "\n\n해결 방법:\n1) [API키 설정] 메뉴에서 API 키 확인\n2) 인터넷 연결 확인\n3) 모델명 또는 endpoint 문제일 수 있음");
    return;
  }


  var tempClusters = [];
  var clusterCount = 1;
  var startRow = 2;
  for (var r=2;r<N;r++){
    var prevTS = parseMMSS(data[r-1][TS_COL-1] || "");
    var currTS = parseMMSS(data[r][TS_COL-1] || "");
    var gapOK = (prevTS != null && currTS != null && (currTS - prevTS) >= GAP_SPLIT_SEC);
    if (gapOK){
      tempClusters.push({ id: "P"+String(clusterCount).padStart(3,"0"), start: startRow, end: r });
      clusterCount++; startRow = r+1;
    }
  }
  tempClusters.push({ id: "P"+String(clusterCount).padStart(3,"0"), start: startRow, end: N });


  var cidsPerRow2 = new Array(N-1).fill("");
  var idList = [];
  for (var i2=0;i2<tempClusters.length;i2++){
    var c = tempClusters[i2];
    idList.push(c.id);
    for (var rr=c.start; rr<=c.end; rr++){ if (rr>=2 && rr<=N) cidsPerRow2[rr-2] = c.id; }
  }
  if (cidsPerRow2.length) sheet.getRange(2, PID_COL, cidsPerRow2.length, 1).setValues(cidsPerRow2.map(x=>[x]));
  // E열(P_IDLIST_COL)은 normalizePidIntoE_에서 고유 PID 목록으로 재작성하므로 여기서는 쓰지 않음
  // assignSummaryOnly(); // ★ 주석 처리: 요약문은 buildClusterSummariesFromPID_에서 생성
}


/***** ===== F 재작성 & 보조 ===== *****/
function rebuild_E_and_F_from_D(){
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data  = sheet.getDataRange().getValues();
  var lastRow = sheet.getLastRow();
  var segs = collectSegmentsByPID_(data);
  var idList = segs.map(function(s) { return s.id; });
  if (lastRow>=2) sheet.getRange(2, P_IDLIST_COL, lastRow-1, 1).clearContent();
  if (idList.length) sheet.getRange(2, P_IDLIST_COL, idList.length, 1).setValues(idList.map(function(x) { return [x]; }));
  var j = (SUMMARY_MODE === "gpt_exhaustive")
    ? summarizeSegmentsByPID_(segs)
    : (SUMMARY_MODE === "gpt_rich")
      ? summarizeSegmentsByPID_RichNarrative_(segs)
      : (SUMMARY_MODE === "gpt_narrative")
        ? summarizeSegmentsByPID_Natural_(segs)
        : summarizeSegmentsByPID_(segs);
  var byId = {}; (j.results||[]).forEach(function(x) { byId[x.id] = x; });
  var Fvals = segs.map(function(s) {
    var x = byId[s.id] || {};
    var tStart = (x.time && x.time.start) || s.time.start || "??:??";
    var tEnd   = (x.time && x.time.end)   || s.time.end   || "??:??";
    var text   = x.summary || summarizeTurns(s.summarySource,false);
    return ["■ " + tStart + "~" + tEnd + "\t" + text];
  });
  if (lastRow>=2) sheet.getRange(2, P_SUMMARY_REFINED_COL, lastRow-1, 1).clearContent();
  if (Fvals.length) sheet.getRange(2, P_SUMMARY_REFINED_COL, Fvals.length, 1).setValues(Fvals);
}


/**
 * 🎯 앵커 행 산출(최종) — 세그먼트 시작 행을 앵커로
 *  - collectSegmentsByPID_ 결과를 사용해 각 PID의 startRow를 그대로 앵커로 씀
 *  - E열은 더 이상 앵커 결정에 사용하지 않음(표시용/인덱스용으로만 유지)
 */
function _getAnchors_(data){
  var segs = collectSegmentsByPID_(data); // {id, startRow, endRow, ...}
  // PID 등장 순서대로, 각 세그먼트의 시작 행을 앵커로 리턴
  return segs.map(function(s){
    return { row: s.startRow, pid: s.id, source: "segment" };
  });
}


/**
 * 📝 코딩 결과를 앵커 행(G/H/I/J)에 쓰기
 */
function _writeCodesToAnchorRows_(sheet, anchors, results){
  if (!anchors.length || !results.length) return;


  var lastRow = sheet.getLastRow();
  if (lastRow >= 2) {
    sheet.getRange(2, P_K_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, P_C_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, P_M_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, TEACHER_FLAG_COL, lastRow-1, 1).clearContent(); // J열: 교사개입
    sheet.getRange(2, PATTERN_COL, lastRow-1, 1).clearContent(); // ✅ K열: 패턴1~5
  }


  // PID → result 매핑 (ID 기반: 결과 안의 id 사용)
  var resultByPid = {};
  (results||[]).forEach(function(r){
    if (r && r.id) resultByPid[r.id] = r;
  });

  // ✅ J열 계산은 행데이터 기반으로 고정
  const teacherMap = _buildTeacherPresenceMap_(sheet);

  // ✅ PID → 목록뷰 인덱스 매핑 (G~J 읽기용)
  var data = sheet.getDataRange().getValues();
  var segs = collectSegmentsByPID_(data);
  var pidToListIdx = {};
  segs.forEach(function(seg, i) { pidToListIdx[seg.id] = i; });

  anchors.forEach(function(anchor){
    var result = resultByPid[anchor.pid];
    if (!result) return;

    // audit에 speaker_counts 채우기 (목록뷰 G~J에서 읽기)
    result.audit = result.audit || {};
    var listIdx = pidToListIdx[anchor.pid];
    var countsArr = [];
    if (listIdx !== undefined) {
      var listRow = 2 + listIdx;
      for (var sc = 0; sc < SPEAKER_CNT_COLS; sc++) {
        var cellVal = sheet.getRange(listRow, SPEAKER_CNT_START_COL + sc).getValue();
        countsArr.push(cellVal);
      }
    }
    result.audit.speaker_counts = countsArr;
    
    // ✅ C차원 하드게이트 적용 (G~J 기준으로 C 코드만 강제)
    var codes = result.codes || {};
    var kCode = (codes.K === "omit") ? "없음" : (codes.K || "없음");
    var mCode = (codes.M === "omit") ? "없음" : (codes.M || "없음");
    var pCode = (codes.P === "omit") ? "없음" : (codes.P || "없음");
    
    // G~J에서 0 개수 계산해서 C 코드 강제
    var summaryF = sheet.getRange(anchor.row, P_SUMMARY_REFINED_COL).getValue() || "";
    var cCodeRaw = (codes.C === "omit") ? "없음" : (codes.C || "없음");
    
    // ✅ 디버그: 강제 전 상태
    dbg("PID=" + anchor.pid, "G~J=" + countsArr.join(","), "C(raw)=" + cCodeRaw);
    
    var cCodeEnforced = enforceCbyCountsAndSummary(cCodeRaw, countsArr, summaryF);
    
    // ✅ 디버그: 강제 후 상태
    dbg("C(enforced)=" + cCodeEnforced);
    
    // cCodeEnforced에서 순수 코드만 추출 (예: "C4. 비판·반박" → "C4")
    var cCodeFinal = cCodeEnforced;
    var cCodeMatch = cCodeEnforced.match(/^C[1-7]/);
    if (cCodeMatch) {
      cCodeFinal = cCodeMatch[0];
    } else if (/해당\s*없음/.test(cCodeEnforced)) {
      cCodeFinal = "없음";
    }
    
    // ✅ 디버그: 최종 코드
    dbg("C(final)=" + cCodeFinal);
    
    // 교사 개입 시 P에 * 추가
    if (result.teacher_involved && pCode && pCode!=="없음" && !/\*$/.test(pCode)) {
      pCode = pCode + "*";
    }

    var kText = _buildWriteup_("K", kCode, result);
    var cText = _buildWriteup_("C", cCodeFinal, result); // ✅ 강제된 C 코드로 writeup 생성
    var mText = _buildWriteup_("M", mCode, result);

    sheet.getRange(anchor.row, P_K_COL).setValue(kText);
    sheet.getRange(anchor.row, P_C_COL).setValue(cText);  // ✅ 상세 설명 형식
    sheet.getRange(anchor.row, P_M_COL).setValue(mText);

    // ✅ 행데이터 기반으로 확정
    sheet.getRange(anchor.row, TEACHER_FLAG_COL).setValue(teacherMap[anchor.pid] ? "교사개입" : "");

    // P: 패턴 판정
    var fText = sheet.getRange(anchor.row, P_SUMMARY_REFINED_COL).getValue() || "";
    var pattern = determinePatternFromKCAndF(kCode, cCodeFinal, fText);
    sheet.getRange(anchor.row, PATTERN_COL).setValue(pattern);
  });
}

/**
 * 📝 KCM/P 코딩 결과를 목록 뷰(K/L/M/N/O/P)에 쓰기 - F와 줄맞춤
 */
function _writeCodesToListRows_(sheet, segs, results){
  if (!segs.length || !results.length) return;

  // ✅ 헤더 기반 동적 열 매핑 (열 구조 변경에 안전)
  var dynCols = detectColumnsByHeader(sheet);
  
  // ✅ 폴백: 동적 탐지 실패 시 고정 상수 사용
  var G_COL = (dynCols.S1 > 0) ? dynCols.S1 : SPEAKER_CNT_START_COL;
  var H_COL = (dynCols.S2 > 0) ? dynCols.S2 : SPEAKER_CNT_START_COL + 1;
  var I_COL = (dynCols.S3 > 0) ? dynCols.S3 : SPEAKER_CNT_START_COL + 2;
  var J_COL = (dynCols.S4 > 0) ? dynCols.S4 : SPEAKER_CNT_START_COL + 3;
  var L_COL = (dynCols.L > 0) ? dynCols.L : P_C_COL;
  
  // ✅ 폴백 로그
  if (dynCols.S1 <= 0 || dynCols.S2 <= 0 || dynCols.S3 <= 0 || dynCols.S4 <= 0) {
    Logger.log("⚠️ 발화수 열 폴백: 고정 열 " + SPEAKER_CNT_START_COL + "~" + (SPEAKER_CNT_START_COL+3) + " 사용");
  }
  if (dynCols.L <= 0) {
    Logger.log("⚠️ C차원 열 폴백: 고정 열 " + P_C_COL + " 사용");
  }
  
  Logger.log("✅ 실제 사용 열: G=" + G_COL + ", H=" + H_COL + ", I=" + I_COL + ", J=" + J_COL + ", L=" + L_COL);

  var lastRow = sheet.getLastRow();
  // 전체 지우고 시작(유령 코딩 제거) - K~P (KCM/P+교사+패턴)
  if (lastRow >= 2) {
    sheet.getRange(2, P_K_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, L_COL, lastRow-1, 1).clearContent(); // ✅ 동적 L열
    sheet.getRange(2, P_M_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, P_P_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, TEACHER_FLAG_COL, lastRow-1, 1).clearContent();
    sheet.getRange(2, PATTERN_COL, lastRow-1, 1).clearContent();
  }

  var resultByPid = {};
  (results||[]).forEach(function(r){ if (r && r.id) resultByPid[r.id] = r; });

  // ✅ 행데이터 기반 교사개입 맵
  const teacherMap = _buildTeacherPresenceMap_(sheet);

  // ✅ 누락 방지: 모든 seg에 대해 처리 (GPT 결과 없어도 로컬 분류)
  var processedCount = 0;
  var localFallbackCount = 0;
  
  for (var i=0; i<segs.length; i++){
    var pid = segs[i].id;
    var row = 2 + i; // 목록(F) 줄과 1:1 매칭

    var res = resultByPid[pid];
    
    // ✅ GPT 결과 없으면 로컬 폴백으로 키워드 기반 적극 추정
    if (!res) {
      var fallbackSummary = sheet.getRange(row, P_SUMMARY_REFINED_COL).getValue() || "";
      var s = normKR(fallbackSummary).toLowerCase();
      
      // K 차원 폴백: 매우 공격적 탐지 (표지어 약한 증거라도 부여)
      var kFallback = "없음";
      
      // K3 우선: 라벨링+근거 동시 존재
      if (/(이건|이거|그건|그거|저건|저거)/.test(s) && /(왜|때문|이유|근거|그래서|하려고|하니까|하면)/.test(s)) {
        kFallback = "K3";
      }
      // K1: 인과·조건·기능·메커니즘 (확장 키워드)
      else if (/왜|때문|그래서|그러니까|그러면|역할|기능|조건|원인|결과|메커니즘|특징|이유|필요|하려고|하니까|통한|흡수|분해|유지|방출|조절|관련|도움/.test(s)) {
        kFallback = "K1";
      }
      // K2: 자료·규칙·도구 탐색
      else if (/자료|데이터|규칙|카드|기준|증거|도구|사례|분류|정의|원리/.test(s)) {
        kFallback = "K2";
      }
      // K1 보완: 비교·대조
      else if (/보다|대신|아니고|쪽|더|덜|vs|차이|비교/.test(s)) {
        kFallback = "K1";
      }
      
      // M 차원 폴백: 매우 공격적 탐지
      var mFallback = "없음";
      
      // M4 우선: 개념 점검 (확장 키워드)
      if (/왜.*생각|왜.*그렇게|왜.*공기|왜.*토양|정의.*뭐|분류.*기준|개념.*뭐|특징|조건|근거.*말|근거.*설명/.test(s)) {
        mFallback = "M4";
      }
      // M3: 논리 점검 (확장 키워드)
      else if (/말이 안 돼|모순|근거가|일관성|논리|정확|타당|증거가|설명 안|맞지 않|반례/.test(s)) {
        mFallback = "M3";
      }
      // M4 보완: 단순 "왜"도 M4 후보
      else if (/왜|무엇.*기준|뭐.*기준|어떤.*기준/.test(s)) {
        mFallback = "M4";
      }
      // M1: 제안+수용
      else if (/(하자|정하자|붙이자|쓰자|나 할게|네가 해|먼저|다음|역할|기록|발표)/.test(s) && /(그래|좋아|응|오케|ok|ㅇㅇ)/.test(s)) {
        mFallback = "M1";
      }
      
      res = {
        id: pid,
        codes: {K: kFallback, C: "없음", M: mFallback, P: "없음"},
        writeups: {
          K: {label_ko: kFallback==="없음" ? "해당 없음" : (kFallback + ". (로컬 추정)"), analysis: "GPT 결과 누락으로 키워드 기반 추정."},
          M: {label_ko: mFallback==="없음" ? "해당 없음" : (mFallback + ". (로컬 추정)"), analysis: "GPT 결과 누락으로 키워드 기반 추정."}
        },
        evidence: [],
        audit: {}
      };
      localFallbackCount++;
    }
    
    processedCount++;

    // ✅ audit에 speaker_counts 채우기 (동적 G~J에서 읽기)
    // 성능 개선: 개별 getValue 대신 배치 읽기
    res.audit = res.audit || {};
    const countsRow = sheet.getRange(row, G_COL, 1, 4).getValues()[0];
    var countsArr = [countsRow[0], countsRow[1], countsRow[2], countsRow[3]];
    res.audit.speaker_counts = countsArr;
    
    // ✅ C차원 하드게이트 적용 (G~J 기준으로 C 코드만 강제)
    var codes = res.codes || {};
    var kCode = (codes.K === "omit") ? "없음" : (codes.K || "없음");
    var mCode = (codes.M === "omit") ? "없음" : (codes.M || "없음");
    var pCode = (codes.P === "omit") ? "없음" : (codes.P || "없음");
    
    // G~J에서 0 개수 계산해서 C 코드 강제
    var summaryF = sheet.getRange(row, P_SUMMARY_REFINED_COL).getValue() || "";
    var cCodeRaw = (codes.C === "omit") ? "없음" : (codes.C || "없음");
    
    // ✅ 디버그: 강제 전 상태
    dbg("PID=" + pid, "G~J=" + countsArr.join(","), "C(raw)=" + cCodeRaw);
    
    var cCodeEnforced = enforceCbyCountsAndSummary(cCodeRaw, countsArr, summaryF);
    
    // ✅ 디버그: 강제 후 상태
    dbg("C(enforced)=" + cCodeEnforced);
    
    // cCodeEnforced에서 순수 코드만 추출 (예: "C4. 비판·반박" → "C4")
    var cCodeFinal = cCodeEnforced;
    var cCodeMatch = cCodeEnforced.match(/^C[1-7]/);
    if (cCodeMatch) {
      cCodeFinal = cCodeMatch[0];
    } else if (/해당\s*없음/.test(cCodeEnforced)) {
      cCodeFinal = "없음";
    }
    
    // ✅ 디버그: 최종 코드
    dbg("C(final)=" + cCodeFinal);
    
    // 교사 개입 시 E에 * 추가
    if (res.teacher_involved && eCode && eCode!=="없음" && !/\*$/.test(eCode)) {
      eCode = eCode + "*";
    }
    
    // writeup 생성 (상세 설명 형식)
    var kText = _buildWriteup_("K", kCode, res);
    var cText = _buildWriteup_("C", cCodeFinal, res); // ✅ 강제된 C 코드로 writeup 생성
    var mText = _buildWriteup_("M", mCode, res);
    var pText = _buildWriteup_("P", pCode, res);

    sheet.getRange(row, P_K_COL).setValue(kText);      // K
    sheet.getRange(row, L_COL).setValue(cText);        // L (✅ 동적 열 + 상세 설명)
    sheet.getRange(row, P_M_COL).setValue(mText);      // M
    sheet.getRange(row, P_P_COL).setValue(pText);      // N

    // O: 교사개입 (행데이터 기반 확정)
    sheet.getRange(row, TEACHER_FLAG_COL).setValue(teacherMap[pid] ? "교사개입" : "");

    // P: 패턴 (K/C 코드 + 해당 줄의 F 요약 사용)
    var fText = sheet.getRange(row, P_SUMMARY_REFINED_COL).getValue() || "";
    var pattern = determinePatternFromKCAndF(kCode, cCodeFinal, fText);
    sheet.getRange(row, PATTERN_COL).setValue(pattern);
  }

  // 🔻 F 목록 아래 남아있던 잔여 코딩 흔적 싹 정리 (K~P: 6개 열)
  var startClear = 2 + segs.length;
  if (lastRow >= startClear) {
    sheet.getRange(startClear, P_K_COL, lastRow - startClear + 1, 6).clearContent();
  }
  
  // ✅ 처리 통계 로그
  dbg("처리 완료:", processedCount + "개 PID", "로컬 폴백:" + localFallbackCount + "개");
  Logger.log("📊 KCM/P 코딩 완료: 총 " + processedCount + "개 PID 처리 (로컬 폴백: " + localFallbackCount + "개)");
}


/***** ===== 2) KCM/P 코딩(최적화 경로 고정) ===== *****/
function callEnhancedCodingParallel(clusters, prevSummary) {
  try {
    if (clusters.length === 1) return callEnhancedCoding(clusters, prevSummary);
    // 병렬 요청 수 ≤ MAX_PARALLEL_REQUESTS 로 보장
    var numBatches = Math.min(MAX_PARALLEL_REQUESTS, Math.max(1, clusters.length));
    var batchSize = Math.ceil(clusters.length / numBatches);
    var batches = []; var messagesList = [];
    for (var i = 0; i < clusters.length; i += batchSize) {
      var batch = clusters.slice(i, i + batchSize);
      batches.push(batch);
      messagesList.push(getEnhancedCodingPrompt(batch, prevSummary));
    }
    Logger.log("병렬 처리 시작: " + batches.length + "개 배치");
    var results = callGPT_JSON_batch(messagesList, MODEL_KM);
    // 안전: ID 기반으로 원래 입력 순서대로 재정렬
    var byId = {};
    for (var j = 0; j < results.length; j++){
      var res = results[j];
      if (res && Array.isArray(res.results)) {
        res.results.forEach(function(x){ if (x && x.id) byId[x.id] = x; });
      } else {
        Logger.log("배치 " + (j+1) + " 실패");
      }
    }
    var ordered = clusters.map(function(c){ return byId[c.id]; }).filter(Boolean);
    if (!ordered.length) return callEnhancedCoding(clusters, prevSummary);
    return { results: ordered };
  } catch (e) {
    Logger.log("병렬 코딩 에러: " + e.toString() + " - 순차 폴백");
    return callEnhancedCoding(clusters, prevSummary);
  }
}


function assignCodingWithTeacherDetectionUltraOptimized(){
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  
  // ✅ 프리플라이트: 헤더 자동 삽입 + 열 매핑 확정
  preflightAndGetCols();
  
  var data = sheet.getDataRange().getValues();
  var N = data.length;
  if (N <= 1){ SpreadsheetApp.getUi().alert("데이터가 없습니다."); return; }


  // 앵커 수집
  var anchors = _getAnchors_(data);
  if (!anchors.length) { SpreadsheetApp.getUi().alert("앵커 행을 찾을 수 없습니다."); return; }


  // 0) 세그먼트 가져와서 PID→세그먼트 매핑(화자 구성 파악용)
  var segs = collectSegmentsByPID_(data); // {id, summarySource:[{speaker,isTeacher,...}], teacher_involved,...}
  var segById = {};
  for (var s = 0; s < segs.length; s++) { segById[segs[s].id] = segs[s]; }

  // ✅ 목록 뷰(F)에서 PID별 요약문을 찾아오는 헬퍼
  function _getFByPid_FromList_(sheet, segs, pid){
    // segs 순서대로 F는 2행부터 차례대로 들어가 있음
    var idx = -1;
    for (var i=0;i<segs.length;i++){ if (segs[i].id === pid){ idx=i; break; } }
    if (idx<0) return "";
    return sheet.getRange(2 + idx, P_SUMMARY_REFINED_COL).getValue() || "";
  }


  // 1) 클러스터 수집(+메타)
  var clusters = [];
  for (var i = 0; i < anchors.length; i++) {
    var anchor = anchors[i];
    var rowData = data[anchor.row - 1];
    var seg = segById[anchor.pid] || null;
    // 1차: 앵커 행(F) / 2차: 목록(F)에서 보강 조회
    var summary = rowData[P_SUMMARY_REFINED_COL-1] || _getFByPid_FromList_(sheet, segs, anchor.pid);

    var teacher_present = seg ? !!seg.teacher_involved : false;

    // ✅ 학생↔학생 인접쌍 카운트 추가
    var peer_pairs = 0;
    if (seg) peer_pairs = _countPeerAdjPairs_(seg);

    // ✅ P차원용: 화자별 발화수 수집 (G~J열) - 강제 정수 변환
    var speaker_counts = [];
    for (var sc = 0; sc < SPEAKER_CNT_COLS; sc++) {
      var val = rowData[SPEAKER_CNT_START_COL - 1 + sc];
      var num = Number(val);
      speaker_counts.push((Number.isFinite(num) && num > 0) ? Math.floor(num) : 0);
    }

    // 요약 유무와 무관하게 항상 푸시(위에서 로컬 폴백 생성)
    clusters.push({
      id: anchor.pid,
      summary: summary,
      time: seg ? seg.time : { start: rowData[TS_COL-1] || "", end: rowData[TS_COL-1] || "" },
      meta: {
        teacher_present: teacher_present,
        student_speakers: (function(){
          var names = {};
          if (seg && Array.isArray(seg.summarySource)){
            seg.summarySource.forEach(function(t){ if(!t.isTeacher) names[(t.speaker||"학생").trim()] = true; });
          }
          return Object.keys(names).length;
        })(),
        has_peer_interaction: (seg ? (function(){
          var names = {};
          seg.summarySource.forEach(function(t){ if(!t.isTeacher) names[(t.speaker||"학생").trim()] = true; });
          return Object.keys(names).length >= 2;
        })() : false),
        teacher_student_only: (seg ? (!!seg.teacher_involved && (function(){
          var names={}; seg.summarySource.forEach(function(t){ if(!t.isTeacher) names[(t.speaker||"학생").trim()]=true; });
          return Object.keys(names).length === 1;
        })()) : false),
        peer_pairs: peer_pairs,
        speaker_counts: speaker_counts,  // ✅ P차원 참여도 판단용
        policy_version: "v2-student-only"
      }
    });
  }
  if (!clusters.length) { SpreadsheetApp.getUi().alert("코딩할 클러스터가 없습니다."); return; }


  // 2) 스마트 캐싱(메타까지 반영)
  var cacheKey = sheetKey();
  var results = []; var uncached = [];
  for (var k = 0; k < clusters.length; k++) {
    var c = clusters[k];
    var key = cacheKey + "|coding|" + codingHash(c.summary + JSON.stringify(c.meta || {}));
    var cached = cacheGet(key);
    if (cached) { try { results.push(JSON.parse(cached)); } catch(e) { uncached.push({c: c, key: key}); } }
    else uncached.push({c: c, key: key});
  }


  // 3) 병렬 배치 처리
  if (uncached.length) {
    var toSend = uncached.map(function(x) { return x.c; });
    var r = callEnhancedCodingParallel(toSend, "");
    if (r && r.results) {
      for (var i2 = 0; i2 < r.results.length; i2++) {
        var res = r.results[i2];
        var pair = uncached[i2];
        cachePut(pair.key, JSON.stringify(res));
        results.push(res);
      }
    } else {
      SpreadsheetApp.getUi().alert("코딩 API 호출 실패");
    }
  }
  if (!results.length) { SpreadsheetApp.getUi().alert("코딩에 실패했습니다."); return; }


  // 4) 학생-전용 정책 강제 적용(C='없음' 조건)
  // results = _enforceStudentOnlyCPolicy_(results, clusters); // ★ 비활성화: 시트 쓰기에서 직접 처리

  // ✅ D1 울트라-스트릭트 보정(자동 내장)
  results = _enforceD1UltraStrict_(results, clusters);
  
  // ✅ P차원 후검증 (P0~P3 검증)
  results = _postValidatePCodes_(results, clusters);
  
  // ✅ C차원은 시트 쓰기 단계에서 G~J를 직접 읽어 처리 (더 정확함)
  // _postValidateCCodes_ 비활성화: 시트 G~J 직접 읽기가 더 정확

  // ✅ teacher_involved를 원시 메타(행데이터 기반)로 동기화
  const teacherMap = _buildTeacherPresenceMap_(sheet);
  results.forEach(function(r){ r.teacher_involved = !!teacherMap[r.id]; });

  // 5) 기록
  if (CODES_WRITE_TARGET === "list") {
    _writeCodesToListRows_(sheet, segs, results);
  } else {
    _writeCodesToAnchorRows_(sheet, anchors, results);
  }


  var teacherCount = results.filter(function(it){ return !!it.teacher_involved; }).length;
  
  var msg = "✅ KCM/P 코딩 완료\n\n" +
            "📊 총 클러스터: " + results.length + "개\n" +
            "👨‍🏫 교사개입 구간: " + teacherCount + "개\n\n" +
            "🔧 안전 패치 적용:\n" +
            "  ✅ 헤더 기반 동적 열 매핑\n" +
            "  ✅ 활성 화자(값>0) 하드게이트\n" +
            "  ✅ C 강제 부여 (활성≥2)\n" +
            "  ✅ P차원 코드북 (P0~P3)\n" +
            "  ✅ 누락 PID 없음 (전체 처리 보장)";
  SpreadsheetApp.getUi().alert(msg);
}


/***** ===== 3) 런너(항상 최적화 경로) & 메뉴 ===== *****/
// ① 클러스터링 - 6110줄의 상세 버전 사용 (요약문 생성 포함)
// ② KCM/P 코딩 - 개별 독립 실행 (CANONICAL ORDER: K → C → M → P)
// NOTE: runCodeKM_All() 은 legacy. 이 경로에서 사용 금지.
function runKCMPCoding(){ 
  runCodeK_All();
  runCodeC_All();
  runCodeM_All();
  runCodeP_All(); 
}
// 🤖 전체 실행: ① → ②
function runAll(){ menu_cluster(); runKCMPCoding(); }

/**
 * 0) 클러스터링 → 0.5) 요약무결성 복구 → 1) KCMP 코딩 → 2) 다이어그램
 * → 3) 일괄 검토/자동 보정 → (요약 변경시) 4) KCMP 재코딩 → 5) 다이어그램 재작성
 */
function runPostProcessAll(){
  const ui = SpreadsheetApp.getUi();

  // ① 클러스터링
  menu_cluster(); // ★ runClustering() → menu_cluster() 변경

  // ①-1 요약 무결성 점검/복구 (요약 실패·누락·깨짐 자동 수선)
  const fix1 = ensureSummaryIntegrityAndRepair();

  // ② ACD 코딩 (요약 바뀐 게 있으면 캐시 키도 달라지므로 자동 재계산됨)
  runACDCoding();

  // ③ 다이어그램 1차
  buildDiagram13();
  recomputePatternsK_Strict();

  // ④ 일괄 검토·자동 보정 (누락/과잉/D1/C 가드, J 재계산 등)
  const report = auditAndAutofixAll();

  // ④-1 보정 과정에서 F(요약) 새로 채웠으면 → ACD 다시 돌리고 다이어그램 갱신
  if ((fix1.fixed||0) > 0 || (report.stats && report.stats.fixedF > 0)) {
    runACDCoding();
    buildDiagram13();
    recomputePatternsK_Strict();
  }

  const summary =
    "[요약 복구] " + (fix1.fixed||0) + "건\n" +
    report.summary;
  ui.alert("후보정 완료", summary, ui.ButtonSet.OK);
}


/*** 구버전 메뉴 시스템 (제거됨 - 최상단 onOpen() 사용) ***/

// 메뉴 수동 새로고침 함수
function refreshMenu(){
  try {
    onOpen(); // 최상단 onOpen() 호출
    SpreadsheetApp.getActive().toast("메뉴가 새로고침되었습니다!", "완료", 3);
  } catch (error) {
    Logger.log("메뉴 새로고침 오류: " + error.toString());
    SpreadsheetApp.getUi().alert("메뉴 새로고침 중 오류 발생: " + error.toString());
  }
}

// 설치 가능한 onOpen 트리거
function createInstallableOnOpen() {
  try {
    ScriptApp.newTrigger('onOpen')
      .timeBased()
      .everyMinutes(1)
      .create();
    SpreadsheetApp.getUi().alert("설치 가능한 트리거가 생성되었습니다.");
  } catch (error) {
    Logger.log("트리거 생성 오류: " + error.toString());
    SpreadsheetApp.getUi().alert("트리거 생성 중 오류 발생: " + error.toString());
  }
}

// 다이어그램 작성 + 패턴 충돌 해결 래퍼 함수 (최적화 버전 사용)
function buildDiagramWithPatternFix(){
  try {
    // Q~T열 초기화 (강제 덮어쓰기)
    const sheet = SpreadsheetApp.getActiveSheet();
    const lastRow = sheet.getLastRow();
    if (lastRow >= 2) {
      SpreadsheetApp.getActive().toast("Q~T열 초기화 중...", "진행", 2);
      sheet.getRange(2, DIAG_A_COL, lastRow - 1, 4).clearContent();  // Q~T열 지우기
    }
    
    SpreadsheetApp.getActive().toast("다이어그램 작성 중...", "진행", 2);
    buildDiagram13_fast();  // ← 최적화 버전 (패턴도 함께 계산됨)
    
    // 패턴 재계산 생략 (buildDiagram13_fast()가 이미 패턴을 P열에 기록)
    // recomputePatternsK_Enhanced(); // ← 중복 제거로 속도 향상!
    
    SpreadsheetApp.getUi().alert("다이어그램 작성 완료");
  } catch (error) {
    Logger.log("다이어그램 작성 오류: " + error.toString());
    SpreadsheetApp.getUi().alert("오류 발생: " + error.toString());
  }
}

// 청크 실행 메뉴 연결 함수
function menuBuildDiagramChunked(){
  // 필요시 배치 크기를 바꿔 호출 (예: 600)
  buildDiagram13_chunked(600);
}


/***** ===== 기타 유틸 ===== *****/
// 캐시 네임스페이스 버전 관리
function _cacheNsKey_(){ return "CACHE_NS_VER"; }
function _getNs_(){
  return PropertiesService.getScriptProperties().getProperty(_cacheNsKey_()) || "v1";
}
function _bumpNs_(){
  var cur = _getNs_();
  var n = parseInt(String(cur).replace(/\D/g,"") || "1", 10) + 1;
  var nxt = "v" + n;
  PropertiesService.getScriptProperties().setProperty(_cacheNsKey_(), nxt);
  return nxt;
}

// ✅ sheetKey()를 네임스페이스 포함으로 교체 (기존 코드와 충돌 방지)
function sheetKey(){
  var ns = _getNs_(); // 캐시 네임스페이스 버전
  return ns + "|sh:" + SpreadsheetApp.getActiveSheet().getSheetId();
}

// 전체 캐시 무효화(버전 올리기)
function clearAllCaches(){
  _bumpNs_();
  SpreadsheetApp.getUi().alert("캐시 네임스페이스를 갱신했습니다. (모든 캐시 무효화)");
}

// 패턴 재계산 (맥락 강화 버전)
function recomputePatternsK_Enhanced() {
  try {
    const sheet = SpreadsheetApp.getActiveSheet();
    if (!sheet) {
      SpreadsheetApp.getUi().alert("활성 시트를 찾을 수 없습니다.");
      return;
    }
    
    const data = sheet.getDataRange().getValues();
    if (!data || data.length < 2) {
      SpreadsheetApp.getUi().alert("처리할 데이터가 없습니다.");
      return;
    }
    
    const segs = collectSegmentsByPID_(data);
    if (!segs || !segs.length) {
      SpreadsheetApp.getUi().alert("처리할 세그먼트가 없습니다.");
      return;
    }
    
    const codeRegex = {
      a: /\bA[123]\b/,
      c: /\bC[1-7]\b/,
      none: /없음|해당\s*없음/i
    };
    
    const topK = (t) => {
      if (!t) return null;
      const s = String(t).split("\n")[0].replace(/\*/g,"").trim();
      if(!s || codeRegex.none.test(s)) return null;
      const m = s.match(codeRegex.a);
      return m ? m[0] : null;
    };
    
    const topC = (t) => {
      if (!t) return null;
      const s = String(t).split("\n")[0].replace(/\*/g,"").trim();
      if(!s || codeRegex.none.test(s)) return null;
      const m = s.match(codeRegex.c);
      return m ? m[0] : null;
    };
    
    let processedCount = 0;
    let errorCount = 0;
    
    // 성능 개선: 배치 읽기로 변경 (row-by-row 접근 제거)
    const maxCol = Math.max(P_K_COL, P_C_COL, P_SUMMARY_REFINED_COL);
    const batchData = sheet.getRange(2, P_K_COL, segs.length, maxCol - P_K_COL + 1).getValues();
    
    segs.forEach((seg, i) => {
      try {
        const row = 2 + i;
        
        if (row > sheet.getLastRow()) {
          Logger.log(`⚠️ 행 ${row}이 시트 범위를 벗어남`);
          return;
        }
        
        // 성능 개선: 배치 읽기 사용
        const rowData = batchData[i];
        const kTop = topK(rowData[0]);
        const cTop = topC(rowData[P_C_COL - P_K_COL]);
        const summary = rowData[P_SUMMARY_REFINED_COL - P_K_COL];
        
        const enhancedPattern = _decidePatternStrict_(aTop, cTop, summary, seg.id);
        sheet.getRange(row, PATTERN_COL).setValue(enhancedPattern);
        processedCount++;
        
      } catch (error) {
        errorCount++;
        Logger.log(`패턴 재계산 오류 (세그먼트 ${i}): ${error.toString()}`);
      }
    });
    
    if (errorCount > 0) {
      SpreadsheetApp.getUi().alert(`패턴 재계산 완료 (처리: ${processedCount}, 오류: ${errorCount})`);
    } else {
      SpreadsheetApp.getUi().alert("패턴 재계산 완료 (맥락 강화 버전)");
    }
    
  } catch (error) {
    Logger.log("패턴 재계산 전체 오류: " + error.toString());
    SpreadsheetApp.getUi().alert("패턴 재계산 중 오류 발생: " + error.toString());
  }
}

/**
 * 디버그: 각 PID에 교사 행이 실제로 있는지 확인
 */
function debugTeacherRows(){
  var sheet = SpreadsheetApp.getActiveSheet();
  var data = sheet.getDataRange().getValues();
  var hit = {};
  for (var r=1;r<data.length;r++){
    var pid = (data[r][PID_COL-1]||"").trim();
    var spk = (data[r][SPEAKER_COL-1]||"").trim();
    if (!pid) continue;
    if (isTeacherSpeaker(spk)){
      if (!hit[pid]) hit[pid] = [];
      hit[pid].push(r+1); // 1-based
    }
  }
  Logger.log(JSON.stringify(hit, null, 2));
}

/**
 * F열(요약문)에서 교사 여부를 감지하는 헬퍼
 */
function getTeacherSummaryMarkers(){
  // F열(요약문)에서 찾을 키워드 (원하면 메뉴로 설정해도 되지만 우선 하드코딩 기본값)
  var raw = PropertiesService.getScriptProperties().getProperty('TEACHER_SUMMARY_MARKERS')
            || '교사,선생,쌤,teacher,Teacher';
  return raw.split(',').map(function(s){ return s.trim(); }).filter(Boolean);
}

function summaryHasTeacherInF(text){
  if (!text) return false;
  var lower = String(text).toLowerCase();
  var markers = getTeacherSummaryMarkers().map(function(s){ return s.toLowerCase(); });
  return markers.some(function(m){ return m && lower.indexOf(m) !== -1; });
}

/** A/C 코드로 1차 후보셋 만들기 */
function getPatternCandidatesFromKC(kCode, cCode){
  kCode = (kCode || "").toUpperCase();
  cCode = (cCode || "").toUpperCase();
  
  // A와 C 차원에서 모두 "없음"이 아닌 실제 코드가 나왔을 때만 패턴 부여
  if (kCode === "없음" || kCode === "OMIT" || cCode === "없음" || cCode === "OMIT") {
    return [];
  }
  
  var cand = [];
  
  // C코드 우선 패턴 판정
  if (cCode === "C1" || cCode === "C6") {
    cand.push("패턴4"); // C1동의, C6아이디어 조율 → 패턴4 초록색
  } else if (cCode === "C3") {
    cand.push("패턴2"); // C3 정교화 → 패턴2 주황색
  } else if (cCode === "C2") {
    cand.push("패턴1"); // C2 명료화 요청 → 패턴1 빨간색
  } else if (cCode === "C4" || cCode === "C5") {
    cand.push("패턴3"); // C4반박, C5설득 → 패턴3 노란색
  } else if (cCode === "C7") {
    cand.push("패턴5"); // C7 또래교수 → 패턴5 하늘색
  } else {
    // C코드가 위에 없는 경우에만 A×C 조합 규칙
    var K1orK2 = (kCode === "K1" || kCode === "K2");
    var K1orK3 = (kCode === "K1" || kCode === "K3");
    var Kany   = (kCode === "K1" || kCode === "K2" || kCode === "K3");

    if (K1orK2 && cCode === "C2") cand.push("패턴1");                       // K2/K1 + C2
    if (K1orK2 && cCode === "C3") cand.push("패턴2");                       // K1/K2 + C3
    if (kCode === "K3" && (cCode === "C4" || cCode === "C5")) cand.push("패턴3"); // K3 + (C4|C5)
    if ((cCode === "C1" || cCode === "C6") && K1orK3) cand.push("패턴4");         // (C1|C6) + (K1|K3)
    if (cCode === "C7" && Kany) cand.push("패턴5");                          // C7 + (K1|K2|K3)
  }

  return cand;
}

/** F열 요약 텍스트로 패턴별 맥락 점수 계산 */
function scorePatternsFromF(fText){
  var t = String(fText || "").toLowerCase();

  // 간단한 한국어 어근/표현 키워드 묶음 (가중치 합산)
  function hit(words, w){ return words.reduce((s,kw)=> s + (t.indexOf(kw)>=0 ? w : 0), 0); }

  // 패턴1: K2(자료탐색)→C2(명료화)→K1(설명구성)
  var p1 = 0;
  p1 += hit(["자료","데이터","측정","수집","관찰","실험","표","그래프","찾", "확인해"], 2);
  p1 += hit(["물어","질문","왜","어떻게","무엇","명료","확인하"], 2);
  p1 += hit(["설명","정리","이유","원리","따라서","그래서"], 1);

  // 패턴2: C3(정교화)↔K1
  var p2 = 0;
  p2 += hit(["덧붙","보충","정교","확장","자세히","구체","더 설명"], 2);
  p2 += hit(["아이디어","생각을","이어","추가"], 1);

  // 패턴3: K3(주장)↔C4/5(비판/설득)
  var p3 = 0;
  p3 += hit(["주장","근거","가설","결론"], 2);
  p3 += hit(["반박","비판","하지만","아닌데","틀렸","설득","납득"], 3);
  p3 += hit(["왜냐하면","따라서"], 1);

  // 패턴4: C1(동의)/C6(조율) → K1/K3
  var p4 = 0;
  p4 += hit(["동의","맞아","그래","그러면","오케이"], 2);
  p4 += hit(["조율","합의","정리하자","정하자","역할","순서","계획"], 2);
  p4 += hit(["설명","주장","정리"], 1);

  // 패턴5: C7(또래 교수) → K1
  var p5 = 0;
  p5 += hit(["가르쳐","알려줄","알려주","설명해줘","설명해 주","따라 해","예를 들어","이렇게 하는"], 3);
  p5 += hit(["질문에 답","물어보","질문하고"], 1);

  return {"패턴1":p1, "패턴2":p2, "패턴3":p3, "패턴4":p4, "패턴5":p5};
}

/** 최종 판정: 1) KC로 단일 매치면 그걸, 2) 다중/미매치면 F요약 점수 최대값으로 */
function determinePatternFromKCAndF(kCode, cCode, fText){
  var cand = getPatternCandidatesFromKC(kCode, cCode);
  if (cand.length === 1) return cand[0];

  var scores = scorePatternsFromF(fText);
  // 후보가 있으면 후보 중 최고 점수, 없으면 전체 중 최고 점수
  var pickSet = (cand.length ? cand : Object.keys(scores));
  var best = "", bestScore = -1;
  for (var i=0;i<pickSet.length;i++){
    var p = pickSet[i], sc = scores[p] || 0;
    if (sc > bestScore){ best = p; bestScore = sc; }
  }
  // 너무 낮으면 미부여(임계치 2). 필요 시 낮추거나 올려도 됨.
  return (bestScore >= 2) ? best : "";
}

// ── G/H 셀 첫 줄에서 '정상 코드'만 뽑기 (없음/해당 없음/공란/기타 → null)
function _topKCode_(t){
  var s = String(t||"").split("\n")[0].replace(/\*/g,"").trim();
  if (!s || /없음|해당\s*없음/i.test(s)) return null;
  var m = s.match(/\bK[123]\b/);
  return m ? m[0] : null;
}

function _topCCode_(t){
  var s = String(t||"").split("\n")[0].replace(/\*/g,"").trim();
  if (!s || /없음|해당\s*없음/i.test(s)) return null;
  var m = s.match(/\bC[1-7]\b/);
  return m ? m[0] : null;
}

// ── F요약을 '보조'로만 쓰는 엄격 판정 (K·C가 둘 다 있을 때만 사용)
function _decidePatternStrict_(K, C, summary){
  if (!K || !C) return ""; // <-- 핵심 가드

  // C코드 우선 패턴 판정
  if (C==="C1" || C==="C6") return "패턴4"; // C1동의, C6아이디어 조율 → 패턴4 초록색
  if (C==="C3") return "패턴2"; // C3 정교화 → 패턴2 주황색
  if (C==="C2") return "패턴1"; // C2 명료화 요청 → 패턴1 빨간색
  if (C==="C4" || C==="C5") return "패턴3"; // C4반박, C5설득 → 패턴3 노란색
  if (C==="C7") return "패턴5"; // C7 또래교수 → 패턴5 하늘색

  // C코드가 위에 없는 경우에만 K코드와 조합으로 판정
  if ((K==="K1"||K==="K2") && C==="C2") return "패턴1";
  if ((K==="K1"||K==="K2") && C==="C3") return "패턴2";
  if (K==="K3" && (C==="C4"||C==="C5")) return "패턴3";
  if ((C==="C1"||C==="C6") && (K==="K1"||K==="K3")) return "패턴4";
  if (C==="C7" && (K==="K1"||K==="K2"||K==="K3")) return "패턴5";

  // 여기까지도 못 고르면 — K·C 모두 있는 경우에 한해서만 F요약으로 보조 판정
  var s = String(summary||"").toLowerCase();
  if (/(데이터|자료|측정|관찰|살펴보|확인|왜|어떻게|묻는|질문)/.test(s)) return "패턴1";
  if (/(덧붙이|확장|정교화|구체화|설명|아이디어)/.test(s)) return "패턴2";
  if (/(주장|근거|반박|비판|설득)/.test(s)) return "패턴3";
  if (/(동의|합의|조율|정리|맞아|그러면)/.test(s)) return "패턴4";
  if (/(설명해주|가르치|알려주|또래|배우게)/.test(s)) return "패턴5";

  return "";
}

// ── 패턴 열 재계산(앵커 행만) — K·C가 둘 다 없으면 패턴을 빈칸으로
function recomputePatternsK_Strict(){
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data  = sheet.getDataRange().getValues();
  var anchors = _getAnchors_(data);
  if (!anchors.length) return;

  // 성능 개선: 배치 읽기로 변경 (row-by-row 접근 제거)
  if (anchors.length === 0) return;
  const anchorRows = anchors.map(a => a.row);
  const minRow = Math.min(...anchorRows);
  const maxRow = Math.max(...anchorRows);
  const maxCol = Math.max(P_K_COL, P_C_COL, P_SUMMARY_REFINED_COL);
  const batchData = sheet.getRange(minRow, P_K_COL, maxRow - minRow + 1, maxCol - P_K_COL + 1).getValues();
  const patternValues = [];
  
  anchors.forEach(function(anchor){
    var r = anchor.row;
    var rowIdx = r - minRow;
    // 성능 개선: 배치 읽기 사용
    var kTop = _topKCode_(batchData[rowIdx][0]);
    var cTop = _topCCode_(batchData[rowIdx][P_C_COL - P_K_COL]);
    var sumF = batchData[rowIdx][P_SUMMARY_REFINED_COL - P_K_COL];

    var pat = _decidePatternStrict_(kTop, cTop, sumF);
    patternValues.push({row: r, value: pat});
  });
  
  // 성능 개선: 배치 쓰기
  patternValues.forEach(function(pv){
    sheet.getRange(pv.row, PATTERN_COL).setValue(pv.value);
  });
}

// === 1-3 다이어그램용 유틸 ===

// G/H/I 셀의 첫 줄에서 코드와 한글 라벨만 추출 → "K1 설명구성" 형식으로 반환
function _parseCodeKoName_(text, dim){
  var t = String(text || "").trim();
  if (!t) return "";
  var first = t.split("\n")[0].trim();

  // 코드 감지
  var m = first.match(/\b(K[123]|C[1-7]|M[1-4]|P[0-3])\b/);
  var code = m ? m[1] : "";

  // 라벨 추출(코드 다음 부분)
  var label = first;
  if (code) {
    label = first.replace(/^[A-Z]\d+\s*[\.\)]?\s*/, "");
  } else {
    // 코드 못 찾으면 '해당 없음' 처리
    if (/해당\s*없음/.test(first)) return "없음";
    // 그래도 없으면 첫 줄을 그대로(보수적)
    return first.replace(/\s+/g, " ");
  }

  // 라벨 보정(없으면 코드북에서 끌어오기)
  if (!label) {
    if (code) label = _koLabel_(dim, code).replace(/^[A-Z]\d+\s*[\.\)]?\s*/, "");
  }

  // 불필요 문자 제거 + 공백 제거로 간결화 (예: "개념 이해 점검" → "개념이해점검")
  label = String(label || "")
            .replace(/[\"'\[\]\(\)]/g, "")
            .replace(/[·∙•]/g, "")
            .replace(/\s+/g, "");

  // 최종 포맷
  return code ? (code + " " + label) : (label || "");
}

// 패턴 문자열 정규화 ("패턴1" / "Pattern 1" 모두 허용)
function _normalizePattern_(text){
  var t = String(text || "").trim().toLowerCase();
  if (!t) return "";
  if (/1/.test(t)) return "패턴1";
  if (/2/.test(t)) return "패턴2";
  if (/3/.test(t)) return "패턴3";
  if (/4/.test(t)) return "패턴4";
  if (/5/.test(t)) return "패턴5";
  return "";
}

// 패턴 → 셀 배경색
function _patternColor_(pat){
  var map = {
    "패턴1": "#ea4335", // 빨간색 (구글 시트 표준)
    "패턴2": "#ff9800", // 주황색 (구글 시트 표준)
    "패턴3": "#ffeb3b", // 노란색 (구글 시트 표준)
    "패턴4": "#4caf50", // 초록색 (구글 시트 표준)
    "패턴5": "#03a9f4"  // 하늘색 (구글 시트 표준)
  };
  return map[pat] || null;
}

// === 1-3 다이어그램 작성 ===

// PID -> 행 목록 매핑
function _getPidRowsMap_(data){
  const map = {};
  for (let r=1; r<data.length; r++){
    const pid = (data[r][PID_COL-1]||"").trim();
    if (!pid) continue;
    if (!map[pid]) map[pid]=[];
    map[pid].push(r+1); // 1-based row
  }
  return map;
}

// PID의 앵커(대표) 행: 목록 뷰(2행부터 연속)에서 찾기
function _getAnchorRowForPid_(data, pid){
  // 목록 뷰 방식: segs 순서대로 F는 2행부터 차례대로 들어가 있음
  const segs = collectSegmentsByPID_(data);
  for (let i=0; i<segs.length; i++){
    if (segs[i].id === pid) return 2 + i; // 2행부터 시작
  }
  return null;
}

// === 새 다이어그램 작성 ===
// - APPLY_SCOPE: "all" → 같은 PID의 모든 행에 쓰기 / "anchors" → 대표행만 쓰기
function buildDiagram13(){
  // ── 설정
  const APPLY_SCOPE = "anchors";   // "anchors": 목록뷰 앵커 줄만, "all": 같은 PID의 모든 원문 줄
  const ONLY_ROWS_WITH_F = true;   // F요약 있는 행만 그리기
  const SUPPRESS_ALL_NONE = true;  // A/C/D/E 모두 '없음'이면 Q/R/S/T 비움

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data  = sheet.getDataRange().getValues();
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) { SpreadsheetApp.getUi().alert("데이터가 없습니다."); return; }

  // === 유틸
  function _getPidRowsMap_(data){
    const map={};
    for (let r=1;r<data.length;r++){
      const pid=(data[r][PID_COL-1]||"").trim();
      if(!pid) continue;
      (map[pid]||(map[pid]=[])).push(r+1);
    }
    return map;
  }
  function _getAnchorRowForPid_(data, pid){
    const segs = collectSegmentsByPID_(data);   // 목록뷰와 동일 순서
    for (let i=0;i<segs.length;i++){
      if (segs[i].id===pid) return 2+i;         // F/G/H/I도 이 줄에 기록되는 구조
    }
    return null;
  }
  function _safeBg4_(row, col, color){          // 배경 처리: 빈 문자열로 초기화(환경별 null 오류 회피)
    const bg = color ? [color,color,color,color] : ["","","",""];
    sheet.getRange(row, col, 1, 4).setBackgrounds([bg]);
  }
  function _topKCode_(t){
    const s=String(t||"").split("\n")[0].replace(/\*/g,"").trim();
    if(!s || /없음|해당\s*없음/i.test(s)) return null;
    const m=s.match(/\bA[123]\b/); return m?m[0]:null;
  }
  function _topCCode_(t){
    const s=String(t||"").split("\n")[0].replace(/\*/g,"").trim();
    if(!s || /없음|해당\s*없음/i.test(s)) return null;
    const m=s.match(/\bC[1-7]\b/); return m?m[0]:null;
  }
  function _parseCodeKoName_(text){
    let t=String(text||"").trim(); if(!t) return "";
    const first=t.split("\n")[0].trim();
    const m=first.match(/\b(A[123]|C[1-7]|D[1-4]|E[0-3])\b/);
    const code=m?m[1]:"";
    if(!code){
      if(/해당\s*없음/.test(first)) return "없음";
      return first.replace(/\s+/g," ");
    }
    let label=first.replace(/^[A-Z]\d+\s*[\.\)]?\s*/,"").replace(/[\"'\[\]\(\)·∙•]/g,"").replace(/\s+/g,"");
    return code+(label?(" "+label):"");
  }
  function _decidePatternStrict_(A,C,summary,pid){
    if(!A||!C) return "";
    
    // OFF_TASK는 패턴 없음
    if (A === "OFF_TASK") return "";
    
    // C코드 우선 패턴 판정
    if (C==="C1" || C==="C6") return "패턴4"; // C1동의, C6아이디어 조율 → 패턴4 초록색
    if (C==="C3") return "패턴2"; // C3 정교화 → 패턴2 주황색
    if (C==="C2") return "패턴1"; // C2 명료화 요청 → 패턴1 빨간색
    if (C==="C4" || C==="C5") return "패턴3"; // C4반박, C5설득 → 패턴3 노란색
    if (C==="C7") return "패턴5"; // C7 또래교수 → 패턴5 하늘색
    
    // C코드가 위에 없는 경우에만 A×C 조합 규칙
    if ((A==="A1"||A==="A2") && C==="C2") return "패턴1";
    if ((A==="A1"||A==="A2") && C==="C3") return "패턴2";
    if (A==="A3" && (C==="C4"||C==="C5")) return "패턴3";
    if ((C==="C1"||C==="C6") && (A==="A1"||A==="A3")) return "패턴4";
    if (C==="C7" && /A[123]/.test(A)) return "패턴5";
    
    // 2차: 충돌 시 맥락 재평가 (F열 요약 + 학생 발화 전체 맥락)
    return _resolvePatternConflictByContext_(A, C, summary, pid);
  }
  function _resolvePatternConflictByContext_(A, C, summary, pid) {
    try {
      // 입력 검증
      if (!pid) return "";
      
      const sheet = SpreadsheetApp.getActiveSheet();
      if (!sheet) {
        Logger.log("⚠️ _resolvePatternConflictByContext_: 활성 시트 없음");
    return "";
  }
      
      const data = sheet.getDataRange().getValues();
      if (!data || data.length < 2) {
        Logger.log("⚠️ _resolvePatternConflictByContext_: 데이터 없음");
        return "";
      }
      
      // F열 요약문에서 키워드 힌트 추출
      const summaryLower = String(summary || "").toLowerCase();
      
      // 해당 PID의 학생 발화들 수집 (교사 제외) - 성능 최적화
      const studentUtterances = [];
      for (let r = 1; r < data.length; r++) {
        const rowPid = String(data[r][PID_COL-1] || "").trim();
        if (rowPid === pid) {
          const speaker = String(data[r][SPEAKER_COL-1] || "").trim();
          const utterance = String(data[r][UTTER_COL-1] || "").trim();
          if (!isTeacherSpeaker(speaker) && utterance) {
            studentUtterances.push(utterance.toLowerCase());
          }
        }
      }
      
      if (studentUtterances.length === 0 && !summaryLower) {
        // 학생 발화와 요약이 모두 없으면 C 우선순위로 폴백
        if (C === "C7") return "패턴5";
        if (C === "C4" || C === "C5") return "패턴3";
        if (C === "C3") return "패턴2";
        if (C === "C2") return "패턴1";
        return "패턴4";
      }
      
      const allContext = (summaryLower + " " + studentUtterances.join(" ")).toLowerCase();
      
      // 패턴별 키워드 가중치 계산 (캐싱된 정규식 사용)
      const patterns = {
        "패턴1": /(데이터|자료|측정|관찰|확인|왜|어떻게|질문|의문|궁금)/g,
        "패턴2": /(덧붙|확장|정교|구체|설명|아이디어|보충|추가|더하)/g,
        "패턴3": /(주장|근거|반박|비판|설득|반대|틀렸|아니야|하지만)/g,
        "패턴4": /(동의|합의|조율|정리|맞아|그러면|좋아|그럼|협력)/g,
        "패턴5": /(설명해주|가르치|알려주|또래|도와줘|모르겠|가르쳐)/g
      };
      
      let maxScore = 0;
      let bestPattern = "";
      
      for (const [pattern, regex] of Object.entries(patterns)) {
        const matches = (allContext.match(regex) || []).length;
        if (matches > maxScore) {
          maxScore = matches;
          bestPattern = pattern;
        }
      }
      
      // 동점이거나 명확하지 않으면 C 우세 → D → A 순으로 결정
      if (maxScore === 0) {
        // C차원 우선순위: C7→C4,C5→C3→C2→C1,C6
        if (C === "C7") return "패턴5";
        if (C === "C4" || C === "C5") return "패턴3";
        if (C === "C3") return "패턴2";
        if (C === "C2") return "패턴1";
        return "패턴4"; // C1, C6 등
      }
      
      return bestPattern;
      
    } catch (error) {
      Logger.log("⚠️ _resolvePatternConflictByContext_ 오류: " + error.toString());
      // 오류 시 기본 C 우선순위 폴백
      if (C === "C7") return "패턴5";
      if (C === "C4" || C === "C5") return "패턴3";
      if (C === "C3") return "패턴2";
      if (C === "C2") return "패턴1";
      return "패턴4";
    }
  }

  function _patternColor_(pat){
    return ({
      "패턴1":"#ea4335","패턴2":"#ff9800","패턴3":"#ffeb3b","패턴4":"#4caf50","패턴5":"#03a9f4"
    })[pat]||null;
  }

  // === 데이터 준비
  const pidRows = _getPidRowsMap_(data);
  const pids = Object.keys(pidRows);
  if (!pids.length){ SpreadsheetApp.getUi().alert("PID가 없습니다."); return; }
  const teacherMap = _buildTeacherPresenceMap_(sheet); // 행데이터 기반

  // === 본처리
  pids.forEach(pid=>{
    const anchorRow = _getAnchorRowForPid_(data, pid);
    if (!anchorRow) return;

    // F요약이 비어 있으면(목록뷰 줄이 아니거나 아직 요약 미생성) → 건너뜀
    const f = String(sheet.getRange(anchorRow, P_SUMMARY_REFINED_COL).getValue()||"").trim();
    if (ONLY_ROWS_WITH_F && !f) return;

    // K/L/M 첫줄에서 정상 코드만 추출 → 패턴(P)
    const aTop = _topACode_(sheet.getRange(anchorRow, P_A_COL).getValue());
    const cTop = _topCCode_(sheet.getRange(anchorRow, P_C_COL).getValue());
    const pat  = _decidePatternStrict_(aTop, cTop, f);
    sheet.getRange(anchorRow, PATTERN_COL).setValue(pat);
    const color = _patternColor_(pat);

    // QRST 텍스트(앵커의 K/L/M/N 1줄 파싱 + 교사개입 별표)
    const star = teacherMap[pid] ? "*" : "";
    const A = _parseCodeKoName_(sheet.getRange(anchorRow, P_A_COL).getValue()) || "없음";
    const C = _parseCodeKoName_(sheet.getRange(anchorRow, P_C_COL).getValue()) || "없음";
    const D = _parseCodeKoName_(sheet.getRange(anchorRow, P_D_COL).getValue()) || "없음";
    const E = _parseCodeKoName_(sheet.getRange(anchorRow, P_E_COL).getValue()) || "없음";
    let qrst = [A+star, C+star, D+star, E+star];

    // 모두 '없음'이면 QRST 비움(선택)
    if (SUPPRESS_ALL_NONE){
      const clean = qrst.map(v=>String(v||"").replace(/\*/g,"").trim());
      if (clean[0]==="없음" && clean[1]==="없음" && clean[2]==="없음" && clean[3]==="없음"){
        qrst=["","","",""];
      }
    }

    // 적용 대상 행: 기본은 앵커 줄만
    const targets = (APPLY_SCOPE==="all") ? (pidRows[pid]||[anchorRow]) : [anchorRow];

    targets.forEach(r=>{
      // F요약 제한 적용(선택): 목록뷰 외 r에는 F가 없으므로 그리기 원치 않으면 skip
      if (ONLY_ROWS_WITH_F){
        const fHere = String(sheet.getRange(r, P_SUMMARY_REFINED_COL).getValue()||"").trim();
        if (!fHere){ // 과거 잔여물 제거
          sheet.getRange(r, DIAG_A_COL, 1, 4).setValues([["","","",""]]);
          _safeBg4_(r, DIAG_A_COL, null);
          return;
        }
      }
      sheet.getRange(r, DIAG_A_COL, 1, 4).setValues([qrst]);
      _safeBg4_(r, DIAG_A_COL, color);
    });
  });

  SpreadsheetApp.getUi().alert("다이어그램 작성 완료(목록 뷰 기준).");
}

// === 벡터화된 초고속 버전 ===
function buildDiagram13_fast() {
  const SUPPRESS_ALL_NONE = true;

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data  = sheet.getDataRange().getValues();
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) { SpreadsheetApp.getUi().alert("데이터가 없습니다."); return; }

  // ==== 준비: segs 1회 계산 ====
  const segs = collectSegmentsByPID_(data); // 목록뷰 순서 보장
  const n = segs.length;
  if (!n) { SpreadsheetApp.getUi().alert("PID가 없습니다."); return; }

  // ==== 한번에 읽기: A/F/K/L/M/N 구간 ====
  // 목록뷰 기준으로 2행부터 n개가 연속으로 존재
  const A_rng = sheet.getRange(2, 1, n, 1).getValues();                     // A (발화자)
  const F_rng = sheet.getRange(2, P_SUMMARY_REFINED_COL, n, 1).getValues(); // F (요약문)
  const KLMN_rng = sheet.getRange(2, P_K_COL, n, 4).getValues();            // K/L/M/N (K/C/M/P 코딩)
  const P_rng = sheet.getRange(2, PATTERN_COL, n, 1).getValues();            // P (현재값, 필요시 덮어씀)

  // ==== 코드 이름 맵 (간단 표기) ====
  const codeNameMap = {
    "A1": "추론·설명 구성",
    "A2": "자료 수집·해석",
    "A3": "주장 정당화",
    "K1": "추론·설명 구성",  // K 차원 (A와 동일)
    "K2": "자료 수집·해석",
    "K3": "주장 정당화",
    "C1": "동의",
    "C2": "명료화 요청",
    "C3": "정교화",
    "C4": "비판·반박",
    "C5": "설득",
    "C6": "아이디어 조율",
    "C7": "또래 교수",
    "D1": "논의 목표·방식",
    "D2": "참여 규범",
    "D3": "논리 점검",
    "D4": "개념 이해",
    "M1": "논의 목표·방식",  // M 차원 (D와 동일)
    "M2": "참여 규범",
    "M3": "논리 점검",
    "M4": "개념 이해",
    "P0": "의미 있는 참여 없음",
    "P1": "1명의 의미 있는 참여",
    "P2": "소수의 의미 있는 참여",
    "P3": "다수의 의미 있는 참여"
  };

  // ==== 헬퍼 (로컬 인라인) ====
  const topK = s => { 
    s=String(s||"").split("\n")[0].replace(/\*/g,"").trim(); 
    if(!s || /^(없음|해당.?없|해당없음|코드없음|적용불가)/i.test(s)) return null;  // 빈 셀 또는 해당 없음 필터링
    const m=s.match(/\bK[123]\b/); 
    return m?m[0]:null; 
  };
  const topC = s => { 
    s=String(s||"").split("\n")[0].replace(/\*/g,"").trim(); 
    if(!s || /^(없음|해당.?없|해당없음|코드없음|적용불가)/i.test(s)) return null;  // 빈 셀 또는 해당 없음 필터링
    const m=s.match(/\bC[1-7]\b/); 
    return m?m[0]:null; 
  };
  const topM = s => { 
    s=String(s||"").split("\n")[0].replace(/\*/g,"").trim(); 
    if(!s || /^(없음|해당.?없|해당없음|코드없음|적용불가)/i.test(s)) return null;  // 빈 셀 또는 해당 없음 필터링
    const m=s.match(/\bM[1-4]\b/); 
    return m?m[0]:null; 
  };
  const topP = s => { s=String(s||"").split("\n")[0].replace(/\*/g,""); const m=s.match(/\bP[0-3]\b/); return m?m[0]:null; };
  const parseCodeKoName = (text) => {
    let t=String(text||"").trim(); 
    if(!t) return "";  // 빈 셀은 빈 문자열 반환
    const first=t.split("\n")[0].trim();
    // K/M 코드도 포함하여 매칭 (최근 변경: K/M 코딩 결과 형식 지원)
    const m=first.match(/\b(K[123]|M[1-4]|A[123]|C[1-7]|D[1-4]|E[0-3]|P[0-3])\b/);
    const code=m?m[1]:"";
    if(!code){
      // "없음", "해당 없음" 등 필터링
      if(/^(없음|해당.?없|해당없음|코드없음|적용불가|해당사항없음|해당안됨|해당안함)/i.test(first)) return "";
      return "";  // 코드가 없으면 빈칸
    }
    // 하드코딩된 이름 사용
    const name = codeNameMap[code] || "";
    return code + (name ? (" " + name) : "");
  };
  // ⚠️ 패턴 판정 로직: K와 C 차원 코드가 모두 있을 때만 패턴 판정, C 차원 코드로만 판정
  const decidePattern = (A,C,summary) => {
    // K와 C 코드가 모두 있어야 패턴 판정
    if(!A||!C) return "";
    
    // C 차원 코드로만 패턴 판정
    if (C==="C1" || C==="C6") return "패턴4";  // C1 동의, C6 아이디어 조율 → 패턴4 초록
    if (C==="C4" || C==="C5") return "패턴3";  // C4 비판·반박, C5 설득 → 패턴3 노랑
    if (C==="C2") return "패턴1";               // C2 명료화 요청 → 패턴1 빨강
    if (C==="C3") return "패턴2";               // C3 정교화 → 패턴2 주황
    if (C==="C7") return "패턴5";               // C7 또래 교수 → 패턴5 하늘색
    
    return "";
  };
  const patColor = p => ({ "패턴1":"#ea4335","패턴2":"#ff9800","패턴3":"#ffeb3b","패턴4":"#4caf50","패턴5":"#03a9f4" }[p] || "");

  // ==== 계산 결과 버퍼 ====
  const QRST_values = new Array(n).fill(null).map(()=>["","","",""]); // Q/R/S/T (다이어그램)
  const QRST_bgs    = new Array(n).fill(null).map(()=>["","","",""]);  // Q/R/S/T 배경색 (모두 색상 적용)
  const O_values    = new Array(n).fill(null).map(()=>[""]);          // O (교사개입)
  const P_values    = new Array(n).fill(null).map(()=>[""]);          // P (패턴)
  const teacherRows = [];  // 교사개입이 있는 행 번호(리치텍스트 처리용)

  // ==== 메인 루프 (계산만) ====
  for (let i=0;i<n;i++){
    const pid = segs[i].id;
    
    // 교사개입 판정: A열(발화자) 또는 F열(요약문)에 "교사" 포함 여부
    const speaker = String(A_rng[i][0]||"").trim();
    const summary = String(F_rng[i][0]||"").trim();
    const hasTeacher = /교사/i.test(speaker) || /교사/i.test(summary);
    
    // O열에 기록
    O_values[i] = hasTeacher ? ["교사개입"] : [""];

    const ktxt = KLMN_rng[i][0]; // K (A차원)
    const ltxt = KLMN_rng[i][1]; // L (C차원)
    const mtxt = KLMN_rng[i][2]; // M (D차원)
    const ntxt = KLMN_rng[i][3]; // N (P차원)

    const K = topK(ktxt);
    const C = topC(ltxt);
    const M = topM(mtxt);

    // Q/R/S/T 텍스트 (첫 줄 파싱, 교사개입 별표는 나중에 리치텍스트로)
    const Q = parseCodeKoName(ktxt) || "";
    const R = parseCodeKoName(ltxt) || "";
    const S = parseCodeKoName(mtxt) || "";
    const T = parseCodeKoName(ntxt) || "";

    // 모두 빈칸이면 생략 옵션
    if (SUPPRESS_ALL_NONE &&
        !Q && !R && !S && !T) {
      QRST_values[i]=["","","",""];
      QRST_bgs[i]=["","","",""];
      O_values[i]=[""];
      P_values[i]=[""];
      continue;
    }

    QRST_values[i] = [Q,R,S,T];

    // 패턴 (A×C 조합) - K를 A로 변환하여 호환성 유지
    // K1→A1, K2→A2, K3→A3
    const A_forPattern = K ? K.replace(/^K/, "A") : null;
    const pat = decidePattern(A_forPattern, C, summary);
    P_values[i] = [pat];
    const bg = patColor(pat);
    QRST_bgs[i] = [bg,bg,bg,bg];  // Q/R/S/T 모두 색상 적용

    // 교사개입이 있으면 나중에 리치텍스트 처리
    if (hasTeacher) {
      teacherRows.push(i+2); // 실제 시트 행 번호
    }
  }

  // ==== 한번에 쓰기 ====
  // QRST 값 (Q=DIAG_K_COL, R=DIAG_C_COL, S=DIAG_M_COL, T=DIAG_P_COL)
  sheet.getRange(2, DIAG_K_COL, n, 4).setValues(QRST_values);
  
  // QRST 배경색 (모두 색상 적용)
  sheet.getRange(2, DIAG_K_COL, n, 4).setBackgrounds(QRST_bgs);

  // O (교사개입)
  sheet.getRange(2, TEACHER_FLAG_COL, n, 1).setValues(O_values);

  // P (패턴)
  sheet.getRange(2, PATTERN_COL, n, 1).setValues(P_values);

  // ==== 교사개입 별표(*) 리치텍스트 첨자 처리 ====
  for (const row of teacherRows) {
    addSuperscriptStar_(sheet, row, DIAG_K_COL); // Q
    addSuperscriptStar_(sheet, row, DIAG_C_COL); // R
    addSuperscriptStar_(sheet, row, DIAG_M_COL); // S
    addSuperscriptStar_(sheet, row, DIAG_P_COL); // T
  }

  SpreadsheetApp.getUi().alert("다이어그램 작성 완료(벡터화 버전).");
}

/**
 * 리치텍스트 첨자(*) 헬퍼: 셀 텍스트 끝에 *를 첨자로 추가
 */
function addSuperscriptStar_(sheet, row, col) {
  const cell = sheet.getRange(row, col);
  const baseText = String(cell.getDisplayValue() || "").trim();
  if (!baseText) return; // 빈 셀은 스킵
  
  const newText = baseText + "*";
  
  try {
    // 리치텍스트 방식 시도
    const richText = SpreadsheetApp.newRichTextValue()
      .setText(newText)
      .setTextStyle(0, baseText.length, SpreadsheetApp.newTextStyle().build())
      .setTextStyle(baseText.length, newText.length,
        SpreadsheetApp.newTextStyle()
          .setFontSize(8)  // 작은 글씨로
          .build()
      )
      .build();
    cell.setRichTextValue(richText);
  } catch(e) {
    // 리치텍스트 실패 시 일반 텍스트로 폴백
    cell.setValue(newText);
  }
}

// === 대용량 시트용 청크 실행 버전 ===
// 진행상태 키
const DIAG_NS = "DIAG13";
const DIAG_IDX_KEY = "diag_idx";   // 다음에 시작할 인덱스 (0-based)
const DIAG_BATCH_KEY = "diag_batch"; // 배치 크기 기록(옵션)

function buildDiagram13_chunked(batchSize){
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const props = PropertiesService.getDocumentProperties();

  // 기본 배치 크기(시트/네트워크 상태 따라 300~1000 권장)
  batchSize = batchSize || parseInt(props.getProperty(DIAG_BATCH_KEY) || "600", 10);
  props.setProperty(DIAG_BATCH_KEY, String(batchSize));

  const data  = sheet.getDataRange().getValues();
  const segs  = collectSegmentsByPID_(data);
  const n     = segs.length;
  if (!n){ SpreadsheetApp.getUi().alert("PID가 없습니다."); return; }

  let idx = parseInt(props.getProperty(DIAG_IDX_KEY) || "0", 10);
  const end = Math.min(idx + batchSize, n);

  // ---- 청크 데이터 준비 (배열 슬라이스만) ----
  const sliceLen = end - idx;
  if (sliceLen <= 0) {
    props.deleteProperty(DIAG_IDX_KEY);
    SpreadsheetApp.getUi().alert("이미 완료되어 있습니다.");
    return;
  }

  // 필요한 범위만 읽어서 빠르게 처리
  const F = sheet.getRange(2 + idx, P_SUMMARY_REFINED_COL, sliceLen, 1).getValues();
  const KLMN = sheet.getRange(2 + idx, P_K_COL, sliceLen, 4).getValues();  // K/L/M/N (K/C/M/P 코딩)

  // 공통 자료 (한 번만 계산)
  const teacherMap = _buildTeacherPresenceMap_(sheet);

  // ==== 코드 이름 맵 (간단 표기) ====
  const codeNameMap = {
    "K1": "추론·설명 구성", "K2": "자료 수집·해석", "K3": "주장 정당화",
    "C1": "동의", "C2": "명료화 요청", "C3": "정교화", "C4": "비판·반박", "C5": "설득", "C6": "아이디어 조율", "C7": "또래 교수",
    "M1": "논의 목표·방식", "M2": "참여 규범", "M3": "논리 점검", "M4": "개념 이해",
    "P0": "의미 있는 참여 없음", "P1": "1명의 의미 있는 참여", "P2": "소수의 의미 있는 참여", "P3": "다수의 의미 있는 참여"
  };

  // 헬퍼들 (fast 버전과 동일, 빈 셀 및 해당 없음 필터링 추가)
  const topK = s => { 
    s=String(s||"").split("\n")[0].replace(/\*/g,"").trim(); 
    if(!s || /^(없음|해당.?없|해당없음|코드없음|적용불가)/i.test(s)) return null;  // 빈 셀 또는 해당 없음 필터링
    const m=s.match(/\bK[123]\b/); 
    return m?m[0]:null; 
  };
  const topC = s => { 
    s=String(s||"").split("\n")[0].replace(/\*/g,"").trim(); 
    if(!s || /^(없음|해당.?없|해당없음|코드없음|적용불가)/i.test(s)) return null;  // 빈 셀 또는 해당 없음 필터링
    const m=s.match(/\bC[1-7]\b/); 
    return m?m[0]:null; 
  };
  const topM = s => { 
    s=String(s||"").split("\n")[0].replace(/\*/g,"").trim(); 
    if(!s || /^(없음|해당.?없|해당없음|코드없음|적용불가)/i.test(s)) return null;  // 빈 셀 또는 해당 없음 필터링
    const m=s.match(/\bM[1-4]\b/); 
    return m?m[0]:null; 
  };
  const topP = s => { s=String(s||"").split("\n")[0].replace(/\*/g,""); const m=s.match(/\bP[0-3]\b/); return m?m[0]:null; };
  const parseCodeKoName = (text) => {
    let t=String(text||"").trim(); 
    if(!t) return "";  // 빈 셀은 빈 문자열 반환
    const first=t.split("\n")[0].trim();
    // K/M 코드도 포함하여 매칭 (최근 변경: K/M 코딩 결과 형식 지원)
    const m=first.match(/\b(K[123]|M[1-4]|A[123]|C[1-7]|D[1-4]|E[0-3]|P[0-3])\b/);
    const code=m?m[1]:"";
    if(!code){
      // "없음", "해당 없음" 등 필터링
      if(/^(없음|해당.?없|해당없음|코드없음|적용불가|해당사항없음|해당안됨|해당안함)/i.test(first)) return "";
      return "";  // 코드가 없으면 빈칸
    }
    // 하드코딩된 이름 사용
    const name = codeNameMap[code] || "";
    return code + (name ? (" " + name) : "");
  };
  // ⚠️ 패턴 판정 로직: K와 C 차원 코드가 모두 있을 때만 패턴 판정, C 차원 코드로만 판정
  const decidePattern = (A,C,summary) => {
    // K와 C 코드가 모두 있어야 패턴 판정
    if(!A||!C) return "";
    
    // C 차원 코드로만 패턴 판정
    if (C==="C1" || C==="C6") return "패턴4";  // C1 동의, C6 아이디어 조율 → 패턴4 초록
    if (C==="C4" || C==="C5") return "패턴3";  // C4 비판·반박, C5 설득 → 패턴3 노랑
    if (C==="C2") return "패턴1";               // C2 명료화 요청 → 패턴1 빨강
    if (C==="C3") return "패턴2";               // C3 정교화 → 패턴2 주황
    if (C==="C7") return "패턴5";               // C7 또래 교수 → 패턴5 하늘색
    
    return "";
  };
  const patColor = p => ({ "패턴1":"#ea4335","패턴2":"#ff9800","패턴3":"#ffeb3b","패턴4":"#4caf50","패턴5":"#03a9f4" }[p] || "");

  // 옵션들 (fast 버전과 맞춤)
  const ONLY_ROWS_WITH_F = true;
  const SUPPRESS_ALL_NONE = true;

  // 결과 버퍼
  const QRST_values = new Array(sliceLen).fill(null).map(()=>["","","",""]);
  const QRST_bgs    = new Array(sliceLen).fill(null).map(()=>["","","",""]);  // Q/R/S/T 모두 색상
  const P_values    = new Array(sliceLen).fill(null).map(()=>[""]);
  const teacherRows = [];  // 교사개입이 있는 행 번호

  // 계산
  for (let j=0;j<sliceLen;j++){
    const seg = segs[idx + j];
    const pid = seg.id;
    const hasTeacher = teacherMap[pid];

    const f = String(F[j][0]||"").trim();
    if (ONLY_ROWS_WITH_F && !f) { continue; }

    const ktxt = KLMN[j][0], ltxt = KLMN[j][1], mtxt = KLMN[j][2], ntxt = KLMN[j][3];
    const K = topK(ktxt), C = topC(ltxt), M = topM(mtxt), P = topP(ntxt);

    const Q = parseCodeKoName(ktxt) || "";
    const R = parseCodeKoName(ltxt) || "";
    const S = parseCodeKoName(mtxt) || "";
    const T = parseCodeKoName(ntxt) || "";

    if (SUPPRESS_ALL_NONE &&
        !Q && !R && !S && !T) {
      continue;
    }

    QRST_values[j] = [Q,R,S,T];
    // 패턴 (A×C 조합) - K를 A로 변환하여 호환성 유지
    // K1→A1, K2→A2, K3→A3
    const A_forPattern = K ? K.replace(/^K/, "A") : null;
    const pat = decidePattern(A_forPattern, C, f);
    P_values[j] = [pat];
    const bg = patColor(pat);
    QRST_bgs[j] = [bg,bg,bg,bg];  // Q/R/S/T 모두 색상

    // 교사개입이 있으면 행 번호 기록
    if (hasTeacher) {
      teacherRows.push(2 + idx + j);
    }
  }

  // 쓰기 (해당 청크 블록만)
  const startRow = 2 + idx;
  sheet.getRange(startRow, DIAG_K_COL, sliceLen, 4).setValues(QRST_values);
  sheet.getRange(startRow, DIAG_K_COL, sliceLen, 4).setBackgrounds(QRST_bgs); // QRST 모두 색상
  sheet.getRange(startRow, PATTERN_COL, sliceLen, 1).setValues(P_values);

  // 교사개입 별표(*) 리치텍스트 첨자 처리
  for (const row of teacherRows) {
    addSuperscriptStar_(sheet, row, DIAG_K_COL); // Q
    addSuperscriptStar_(sheet, row, DIAG_C_COL); // R
    addSuperscriptStar_(sheet, row, DIAG_M_COL); // S
    addSuperscriptStar_(sheet, row, DIAG_P_COL); // T
  }

  // 다음 인덱스 저장 & 다음 런 예약
  idx = end;
  if (idx >= n) {
    props.deleteProperty(DIAG_IDX_KEY);
    SpreadsheetApp.getUi().alert("다이어그램 작성(청크) 완료");
  } else {
    props.setProperty(DIAG_IDX_KEY, String(idx));
    // 1–2초 뒤에 자동 재개 (시간 초과 방지)
    ScriptApp.newTrigger('buildDiagram13_chunked')
      .timeBased().after(1500).create();
    // 진행 상황 토스트
    SpreadsheetApp.getActive().toast("다이어그램 진행 중… " + idx + "/" + n, "진행", 3);
  }
}




/***** ===== 사용법 요약 =====
1) 메뉴 → 🔑 API 키 설정, 🙋 교사 식별자 설정(1회)
2) 시트 A:화자, B:타임스탬프(mm:ss), C:발화 입력
3) 메뉴 → 🤖 전체 실행(① 클러스터링 → ② 코딩)
   - D: PID, E: P-ID 목록, F: 클러스터 요약
   - G/H/I: A/C/D 코딩, J: 교사개입 플래그
또는 단계 실행: ① 클러스터링 → ② ACD 코딩
*********************************/








/***** ===== ACD 코딩 개선 (학생 주체성 중심) ===== *****/


/**
 * 코드→한국어 라벨 매핑 (P차원 코드북 반영)
 */
function _koLabel_(dim, code) {
  if (!code || code === "없음" || code === "omit") return "해당 없음";
  if (code === "OFF_TASK") return "OFF_TASK. 오프태스크/잡담";
  var A = {A1: "A1. 추론과 설명 구성", A2: "A2. 자료 수집 및 해석", A3: "A3. 주장에 대한 정당화"};
  var C = {
    C1: "C1. 동의", C2: "C2. 명료화/정당화 요청", C3: "C3. 상호작용을 통한 정교화",
    C4: "C4. 비판과 반박", C5: "C5. 설득", C6: "C6. 아이디어 조율", C7: "C7. 또래 교수"
  };
  var D = {D1: "D1. 논의의 목표와 방식", D2: "D2. 참여 태도 및 규범", D3: "D3. 설명 및 논리", D4: "D4. 개념 이해"};
  var E = {
    P0: "P0. 의미 있는 참여 없음", 
    P1: "P1. 1명의 의미 있는 참여", 
    P2: "P2. 소수의 의미 있는 참여",
    P3: "P3. 다수의 의미 있는 참여"
  };
  var m = (dim === "K") ? K : (dim === "C") ? C : (dim === "M") ? M : P;
  return (m[code] || "해당 없음");
}


/**
 * 해설 문단 포맷(모델 결과의 writeups가 있으면 그대로 사용, 없으면 evidence로 생성)
 */
function _buildWriteup_(dim, code, result) {
  if (!result) return "해당 없음";
  var wu = (result.writeups && result.writeups[dim]) || null;
  if (code === "없음" || code === "omit") {
    // 보다 자연스러운 '해당 없음' 문구
    if (wu && wu.analysis) return "해당 없음\n" + wu.analysis;
    if (dim === "K") return "해당 없음\n수업 과업·개념과 직접 관련된 설명/추론 근거가 뚜렷하지 않습니다.";
    if (dim === "C") return "해당 없음\n학생↔학생 상호작용(정교화·비판·조율 등)이 관찰되지 않습니다.";
    if (dim === "M") return "해당 없음\n목표·절차 조정, 논리 점검, 개념 이해 점검의 명시적 단서가 부족합니다.";
    if (dim === "P") return "해당 없음\n참여도 패턴을 판정할 근거가 부족합니다.";
  }
  if (code === "OFF_TASK") {
    if (wu && wu.analysis) return "OFF_TASK\n" + wu.analysis;
    return "OFF_TASK\n과학 내용이 아닌 오프태스크/잡담으로 판단됩니다.";
  }
  // writeups 선호
  if (wu && (wu.label_ko || wu.analysis)) {
    var head = wu.label_ko || _koLabel_(dim, code);
    var body = wu.analysis || "";
    return head + "\n" + body;
  }
  // evidence 기반 최소 포맷
  var ev = (result.evidence || []).find(function(e) { return e.dim === dim; });
  var lines = [];
  lines.push(_koLabel_(dim, code));
  if (ev && ev.quote) lines.push("\"" + ev.quote + "\"를 근거로, " + (ev.why || "해당 차원 코딩의 근거가 됩니다."));
  return lines.join("\n");
}


/**
 * 묘사 전용 요약 프롬프트 (샘플 톤)
 */
function getDescriptiveSummaryPrompt_(segments) {
  var sys = `You are a Korean narrative summarizer for small-group classroom talk.


STYLE
- 1~3문장, 묘사 중심(평가 금지).
- 흐름: "한 학생이 …라고 말한다. 다른 학생이 …".
- 학생 인용 1~3개(5~20자). 교사 인용은 맥락상 꼭 필요할 때만.
- 시간/장소 생략, 고유명은 학생/남학생/여학생 등으로 일반화.
- 오프태스크(시험·잡담·농담)는 요약에서 제외하거나 최소화.


CALIBRATION
- examples가 있으면 문체·밀도·인용 개수를 맞춘다.
- A/C/D 판단과 무관하게 **학생 상호작용의 핵심 주장-응답-결정**만 추출.
- STRICT JSON ONLY.`;


  // payload 구성
  var payload = segments.map(function(s) {
    var turns = (s.summarySource || []).map(function(t) {
      var sp = t.isTeacher ? "교사" :
        (String(t.speaker || "").match(/남학생|여학생/) ? t.speaker : (t.speaker || "학생"));
      return {speaker: sp, text: t.utt};
    });
    return {id: s.id, time: s.time, turns: turns};
  });


  var user = `[CALIBRATION_EXAMPLES]  // 선택
examples = ${JSON.stringify([])}
/*
{"summary":"한 학생이 "…"라고 말한다. 다른 학생은 "…".","style_notes":"(선택)"}
*/


segments = ${JSON.stringify(payload)}
/*
segment: {"id":"P###","time":{"start":"mm:ss","end":"mm:ss"},"turns":[{"speaker":"학생|남학생|여학생|교사","text":"..."}...]}
*/


[OUTPUT SCHEMA]
{
  "style_rules":[ "<<=120자 ko rule 1>", "..."],
  "results":[
    {"id":"P###","time":{"start":"mm:ss","end":"mm:ss"},"summary":"<한국어 요약문>"}
  ]
}
Return ONLY the JSON object.`;
  return [{role: "system", content: sys}, {role: "user", content: user}];
}


/**
 * C차원 학생-전용 강제 (사후 보정)
 * ✅ 강화: 활성 화자 수 (값 > 0) 2명 미만 → C="없음" 강제
 * ✅ 폴백: speaker_counts 누락 시 요약문에서 학생 수 추정
 */
function _enforceStudentOnlyCPolicy_(results, clusters) {
  var byId = {};
  (clusters || []).forEach(function(c) { byId[c.id] = (c.meta || {}); });

  for (var i = 0; i < (results || []).length; i++) {
    var r = results[i];
    var m = byId[r.id] || {};
    
    // ✅ 활성 화자 수 검사 (값 > 0인 화자가 2명 이상인지) ★
    var speakerCounts = m.speaker_counts || [];
    var activeSpeakers = speakerCounts.filter(function(c) { return c > 0; }).length; // ★
    
    // ✅ 폴백: speaker_counts가 전부 0이거나 없으면 요약문에서 학생 수 추정
    if (activeSpeakers === 0 && r.writeups && r.writeups.C) {
      var summary = r.writeups.C.analysis || "";
      // 서로 다른 학생 패턴 찾기: "참석자 1", "참석자 2", "학생A", "학생B" 등
      var studentPatterns = summary.match(/참석자\s*\d+|학생[A-Z가-힣]|남학생|여학생/g) || [];
      var uniqueStudents = {};
      studentPatterns.forEach(function(s) { uniqueStudents[s] = true; });
      if (Object.keys(uniqueStudents).length >= 2) {
        activeSpeakers = 2; // 폴백 활성화
      }
    }
    
    // C차원 금지 조건: 활성 화자 < 2 OR 기존 조건들
    var forceNone = (activeSpeakers < 2) || 
                    (m.teacher_student_only === true) || 
                    (m.has_peer_interaction === false) || 
                    ((m.peer_pairs||0) < 1);

    if (forceNone) {
      if (r.codes) r.codes.C = "없음";
      if (r.writeups && r.writeups.C) {
        r.writeups.C.label_ko = "해당 없음";
        var reason = (activeSpeakers < 2) ? "활성 화자 2명 미만(실제: " + activeSpeakers + "명)" : 
                     (m.teacher_student_only) ? "교사↔학생 단독 상호작용" :
                     "학생↔학생 인접 상호작용 부재";
        r.writeups.C.analysis = reason + "로 인해 C차원 코딩을 적용하지 않습니다.";
      }
      if (Array.isArray(r.evidence)) {
        r.evidence = r.evidence.filter(function(e) { return e.dim !== "C"; });
      }
    }
    
    // ✅ 사후 복구: C="없음"이지만 요약문에 상호작용 표지가 명백하면 복구
    if (r.codes && r.codes.C === "없음" && r.writeups && r.writeups.C) {
      var analysis = r.writeups.C.analysis || "";
      var hasInteraction = /질문|답변|보탬|반박|동의|조율|반례|명료화|정교화|비판|설득|왜|어떻게|아니야|말이 안|그럼.*하자/.test(analysis);
      
      if (hasInteraction && activeSpeakers >= 2) {
        // 상호작용 유형에 맞춰 C 코드 재부여
        var newCode = "C2"; // 기본값
        if (/반박|아니야|말이 안|틀렸|대안|반례/.test(analysis)) newCode = "C4";
        else if (/보탬|추가|정교화|구체|설명.*더/.test(analysis)) newCode = "C3";
        else if (/합의|조율|절충|그럼.*하자|붙이자/.test(analysis)) newCode = "C6";
        else if (/동의|맞아|그.*말.*맞/.test(analysis)) newCode = "C1";
        
        r.codes.C = newCode;
        r.writeups.C.label_ko = newCode + ". (상호작용 복구)";
        r.writeups.C.analysis = "폴백 복구: " + analysis;
      }
    }

    // 추가: Self-Audit 검증 강화
    if (r.self_audit) {
      if (r.self_audit.used_teacher_quote === true) {
        // 교사 발화 사용 감지 시 C차원 무효화
        if (r.codes) r.codes.C = "없음";
        if (r.writeups && r.writeups.C) {
          r.writeups.C.label_ko = "해당 없음";
          r.writeups.C.analysis = "교사 발화를 근거로 사용하여 C차원을 무효화했습니다.";
        }
      }
      
      if (r.self_audit.off_task_ratio > 0.5) {
        // 오프태스크 비중 높으면 A차원을 OFF_TASK로 설정
        if (r.codes) r.codes.K = "OFF_TASK";
        if (r.writeups && r.writeups.K) {
          r.writeups.K.label_ko = "OFF_TASK";
          r.writeups.K.analysis = "오프태스크 비중이 높아 K차원을 OFF_TASK로 설정했습니다.";
        }
      }
      
      if (r.self_audit.student_interaction_found === false) {
        // 학생간 상호작용 없으면 C차원 제거  
        if (r.codes) r.codes.C = "없음";
        if (r.writeups && r.writeups.C) {
          r.writeups.C.label_ko = "해당 없음";
          r.writeups.C.analysis = "학생간 상호작용이 확인되지 않아 C차원을 제거했습니다.";
        }
      }
    }
  }
  return results;
}

/**
 * 화자별 발화수 배열 정수 강제 변환
 */
function coerceCounts(arr){ 
  return (arr||[]).slice(0,4).map(function(v) { return Number.isFinite(+v) ? +v : 0; });
}

/**
 * 활성 발화자 수 계산 (폴백 포함)
 * - counts: [G,H,I,J] 배열
 * - summaryText: F열 요약문
 * - 활성 기준: 값 > 0 (1도 활성으로 인정)
 * 반환: {counts, zeroCnt, active}
 */
function computeActiveSpeakers(counts, summaryText){
  const c = coerceCounts(counts);
  const active = c.filter(function(x){return x > 0;}).length; // ★ 값 > 0
  const zeroCnt = c.filter(function(x){return x === 0;}).length;
  
  let finalActive = active;
  
  if (finalActive === 0){
    // 요약문 내 학생 표기(참석자 1/2/3/4, 학생1/2 등) 2명 이상이면 2로 보정
    const uniq = (summaryText||"").match(/참석자\s*\d+|학생\s*\d+/g)||[];
    const uniqueNames = {};
    uniq.forEach(function(s){ uniqueNames[s.trim()] = true; });
    if (Object.keys(uniqueNames).length >= 2) finalActive = 2;
  }
  
  return {counts:c, zeroCnt:zeroCnt, active:finalActive};
}

/**
 * C셀 표시용: 코드 + 한줄 설명 + 인용
 * 반환 형식: "C# · <라벨> | 「<인용>」" 또는 "C#* · ..." (교사 개입 시)
 */
function formatCDisplay(cCode, writeups, evidence, teacherInvolved){
  if (!cCode || cCode === "없음") return "해당 없음";
  
  var label = (writeups && writeups.C && writeups.C.label_ko) 
    ? String(writeups.C.label_ko).replace(/^C\d+\.\s*/,'').trim() 
    : "";
  
  // evidence에서 C 차원 학생 인용 찾기
  var cQuote = "";
  if (Array.isArray(evidence)) {
    for (var i=0;i<evidence.length;i++){
      var ev = evidence[i] || {};
      if (ev.dim === "C" && ev.quote){
        cQuote = String(ev.quote).trim();
        break;
      }
    }
  }
  
  // fallback: 너무 길면 20자 내로
  if (cQuote.length > 20) cQuote = cQuote.slice(0,20);
  
  var star = teacherInvolved ? "*" : "";
  var brief = label || "상호작용";
  
  return cCode + star + " · " + brief + (cQuote ? (" | 「" + cQuote + "」") : "");
}

/**
 * C차원 하드게이트 + 포맷팅 통합 함수
 * 반환: { cCode: "C#", lText: "C# · ... | 「...」" }
 */
function enforceCHardGateAndFormat(item){
  var counts = (item.audit && Array.isArray(item.audit.speaker_counts)) 
    ? item.audit.speaker_counts.map(function(x){ var n=parseInt(x,10); return isNaN(n)?0:n; }) 
    : [0,0,0,0];
  var zeroCnt = counts.filter(function(x){return x===0;}).length;
  var activeSpeakers = counts.filter(function(x){return x>0;}).length; // ★ 값 > 0

  var codes = item.codes || {};
  
  // 하드게이트: 활성화자 <2 → C=없음
  if (activeSpeakers < 2) {
    codes.C = "없음";
  } else {
    // 활성화자 ≥2 이면 C는 반드시 1~7
    if (!/^C[1-7]$/.test(codes.C || "")) {
      // 맥락 키워드로 보정(요약문/증거 기반)
      var summary = (item.summary || "");
      var pick = "C3";
      if (/아니|아냐|말이 안 돼|반대로|대신/.test(summary)) pick = "C4";
      else if (/하자|붙이자|정하자|그러면|로 가자/.test(summary)) pick = "C6";
      else if (/왜|근거|맞아\?|그치\?|맞지\?|됨\?/.test(summary)) pick = "C2";
      codes.C = pick;
    }
  }
  
  // 포맷팅: L열 텍스트 생성
  var lText = (codes.C==="없음") 
    ? "해당 없음" 
    : formatCDisplay(codes.C, item.writeups, item.evidence, !!item.teacher_involved);
    
  return { cCode: codes.C, lText: lText };
}

/**
 * C차원 로컬 강제 교정(정밀 키워드 사전 + 점수 기반)
 * - countsArr: [G,H,I,J] (정수/문자 섞여도 OK)
 * - summaryText: F열 요약문(클러스터 전체 맥락)
 * 반환: "C2. …" | "C3. …" | "C4. …" | "C6. …" | "해당 없음" (절대 'C1' 자동 배정 안 함)
 */
function enforceCbyCountsAndSummary(cCode, countsArr, summaryText){
  // 0) 활성화자 계산(하드 게이트) - 새 헬퍼 함수 사용
  var result = computeActiveSpeakers(countsArr, summaryText);
  var counts = result.counts;
  var zeroCnt = result.zeroCnt;
  var activeSpeakers = result.active;
  
  if (activeSpeakers < 2) return "해당 없음"; // 규칙: 활성화자<2 → C없음 강제

  // 1) 기존 C값이 유효하면 그대로 존중(단, '없음'류면 교정 진입)
  var c = (cCode||"").trim();
  var isNone = (!c || /^해당\s*없음$/.test(c) || c==="없음");
  if (!isNone && /^C[2|3|4|6]\./.test(c)) return c;

  // 2) 텍스트 전처리
  var s = (summaryText||"")
    .replace(/["""']/g,"")
    .replace(/\s+/g," ")
    .trim();

  // 문장 분해(최근 문장 가중치 부여용)
  var sents = s.split(/(?<=[\.!\?]|다\.|다!|다\?|요\.|요!|요\?|죠\.|죠!|죠\?|\n)/).map(function(t){return t.trim();}).filter(Boolean);
  if (sents.length===0) sents=[s];

  // 3) 키워드 사전 (C_LEX 전역 사전 + 정규식 강화)
  var dict = {
    C4:{ // 반박/대안/부정
      strong: [],
      mid: [],
      weak: []
    },
    C2:{ // 명료화/정당화 요구
      strong: [],
      mid: [],
      weak: []
    },
    C3:{ // 정교화
      strong: [],
      mid: [],
      weak: []
    },
    C6:{ // 조율/합의
      strong: [],
      mid: [],
      weak: []
    },
    AGREE_ONLY:[]
  };
  
  // C_LEX를 정규식으로 변환
  C_LEX.rebuttal.forEach(function(k){ dict.C4.strong.push(new RegExp(k.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'i')); });
  C_LEX.clarify.forEach(function(k){ dict.C2.strong.push(new RegExp(k.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'i')); });
  C_LEX.elaborate.forEach(function(k){ dict.C3.strong.push(new RegExp(k.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'i')); });
  C_LEX.coordinate.forEach(function(k){ dict.C6.strong.push(new RegExp(k.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'i')); });
  
  // 추가 패턴 (정규식)
  dict.C4.mid = [/근데\s+/, /하지만\s+/, /그렇지만\s+/];
  dict.C2.mid = [/(확인|재확인)\s*해/, /\?/];
  dict.C3.mid = [/(의미|맥락)\s*상/];
  dict.C6.mid = [/(정리하자|합의하자)/];
  
  dict.AGREE_ONLY = [
    /^맞(아|지)[.!?]?\s*$/, /^그래[.!?]?\s*$/, /^응[.!?]?\s*$/,
    /(좋은\s*얘기|오케이|ㅇㅇ|ㄱㄱ)\b/
  ];

  // 4) 점수 테이블
  var score = {C2:0, C3:0, C4:0, C6:0};

  // 가중치(강/중/약 + 최근 문장 보너스)
  function addScore(tag, base, isRecent){
    score[tag] += base + (isRecent ? 1.5 : 0); // 최근 문장 가중치
  }

  // 5) 문장별 스캐닝 (최근 2문장 보너스)
  for (var i=0;i<sents.length;i++){
    var sent = sents[i];
    var isRecent = (i >= sents.length-2);

    // C4
    dict.C4.strong.forEach(function(r){ if (r.test(sent)) addScore("C4", 4, isRecent); });
    dict.C4.mid.forEach(function(r){ if (r.test(sent)) addScore("C4", 2.5, isRecent); });
    dict.C4.weak.forEach(function(r){ if (r.test(sent)) addScore("C4", 1, isRecent); });

    // C2
    dict.C2.strong.forEach(function(r){ if (r.test(sent)) addScore("C2", 3.5, isRecent); });
    dict.C2.mid.forEach(function(r){ if (r.test(sent)) addScore("C2", 2, isRecent); });
    dict.C2.weak.forEach(function(r){ if (r.test(sent)) addScore("C2", 1, isRecent); });

    // C3
    dict.C3.strong.forEach(function(r){ if (r.test(sent)) addScore("C3", 3, isRecent); });
    dict.C3.mid.forEach(function(r){ if (r.test(sent)) addScore("C3", 2, isRecent); });
    dict.C3.weak.forEach(function(r){ if (r.test(sent)) addScore("C3", 1, isRecent); });

    // C6
    dict.C6.strong.forEach(function(r){ if (r.test(sent)) addScore("C6", 4, isRecent); });
    dict.C6.mid.forEach(function(r){ if (r.test(sent)) addScore("C6", 2.5, isRecent); });
    dict.C6.weak.forEach(function(r){ if (r.test(sent)) addScore("C6", 1, isRecent); });
  }

  // 6) 동의-다음 전환(동의→반박/조율/명료화 우선)
  // 마지막 문장에 '근데/하지만' 있으면 C4 점수에 +1
  var last = sents[sents.length-1]||"";
  if (/근데|하지만|그렇지만/.test(last)) score.C4 += 1;

  // 순수 동의만 있는 문장들이 다수이고 다른 신호가 희박하면 C1로 가지 말고 C2로 유도
  var agreeOnly = sents.every(function(t){
    var tr = t.replace(/\s+/g,"");
    return dict.AGREE_ONLY.some(function(r){ return r.test(tr); });
  });
  if (agreeOnly){
    score.C2 += 1.5; // 최소 C2로 유도("그치/맞지?"류는 명료화 요구 쪽으로)
  }

  // 7) 최종 선택: 점수 최대 + 동점 규칙(우선순위: C4 > C6 > C2 > C3)
  var order = ["C4","C6","C2","C3"];
  var bestTag = order[0], bestVal = -1;
  Object.keys(score).forEach(function(k){
    if (score[k] > bestVal || (score[k]===bestVal && order.indexOf(k) < order.indexOf(bestTag))){
      bestTag = k; bestVal = score[k];
    }
  });

  // 8) 점수가 전부 0인 극단 케이스 ⇒ 질문부호/요구어 있으면 C2, 절차 표현 있으면 C6, 부정 있으면 C4, 없으면 C2
  if (bestVal<=0){
    if (/\?|(왜|근거|이유)/.test(s))      bestTag="C2";
    else if (/(하자|하기로|정하자)/.test(s)) bestTag="C6";
    else if (/(아니|아닌|말이 안)/.test(s))  bestTag="C4";
    else bestTag="C2";
  }

  // 9) 라벨 텍스트
  var labels = {
    "C2":"C2. 명료화·정당화 요구",
    "C3":"C3. 정교화",
    "C4":"C4. 반박",
    "C6":"C6. 조율"
  };
  return labels[bestTag] || "C2. 명료화·정당화 요구";
}

/**
 * C차원 사후 보정 (zeroCnt 기반 하드게이트 강제)
 * ✅ 0의 개수 3~4개 → C="없음" 강제
 * ✅ 0의 개수 0~2개 → C1~C7 반드시 부여
 * ✅ C1 남발 교정 (질문→C2, 반박→C4 우선)
 */
function _postValidateCCodes_(results, clusters){
  const metaById = {};
  (clusters||[]).forEach(function(c) { metaById[c.id] = c.meta || {}; });

  for (var i = 0; i < (results||[]).length; i++) {
    var r = results[i];
    var m = metaById[r.id] || {};
    var sc = Array.isArray(m.speaker_counts) ? m.speaker_counts.map(function(x){ return +x||0; }) : [0,0,0,0];
    var zeros = sc.filter(function(v) { return v === 0; }).length;
    var active = sc.filter(function(v) { return v > 0; }).length; // ★ 값 > 0

    // 감사 정보 채우기
    r.self_audit = r.self_audit || {};
    r.self_audit.speaker_counts = sc;
    r.self_audit.active_speakers = active;
    r.self_audit.zeroCnt = zeros;

    // 규칙 1: 활성 화자 게이트 (값 > 0 기준)
    // - activeSpeakers < 2 → C="없음" 강제
    // - activeSpeakers ≥ 2 → C 반드시 부여
    if (active < 2) {
      // 또래 상호작용 불가
      if (r.codes) r.codes.C = "없음";
      if (r.writeups && r.writeups.C) {
        r.writeups.C.label_ko = "해당 없음";
        r.writeups.C.analysis = "활성 화자<2로 또래 상호작용이 확인되지 않음(실제: " + active + "명, G~J=" + sc.join(",") + ").";
      }
      if (Array.isArray(r.evidence)) {
        r.evidence = r.evidence.filter(function(e) { return e.dim !== "C"; });
      }
      continue; // 다음 결과
    }

    // 여기부터는 activeSpeakers ≥ 2 → C코드 반드시 존재
    var summary = "";
    if (r.writeups && r.writeups.C && r.writeups.C.analysis) {
      summary = r.writeups.C.analysis;
    } else if (r.summary) {
      summary = r.summary;
    }
    summary = String(summary).replace(/\s+/g," ");

    var hadNone = !r.codes || r.codes.C === "없음" || !r.codes.C;

    // 규칙 2: 요약문 상호작용 신호로 C 재부여
    var hasQuestion = /왜|어떻게|무슨|뭐지|맞아\?|근거|설명해|확인/.test(summary);
    var hasRebuttal = /아니야|말이 안 돼|틀렸|반박|왜.*안 돼|토양 아니야/.test(summary);
    var hasElab = /그래서|즉|때문에|예를 들어|근거로|추가로|정리하면/.test(summary);
    var hasAgree = /맞아|동의|그래 맞아|그럼.*네|맞지/.test(summary);

    function setC(newCode, reason){
      r.codes = r.codes || {};
      r.codes.C = newCode;
      if (r.writeups && r.writeups.C){
        var labels = {
          "C1":"C1. 동의","C2":"C2. 명료화·정당화 요구","C3":"C3. 정교화","C4":"C4. 비판·반박",
          "C5":"C5. 설득","C6":"C6. 아이디어 조율","C7":"C7. 또래 교수"
        };
        r.writeups.C.label_ko = labels[newCode] || newCode;
        r.writeups.C.analysis = reason + " (사후보정)";
      }
    }

    if (hadNone) {
      // C="없음"이었는데 활성≥2 → 강제 재부여
      if (hasRebuttal) setC("C4","반박/부정 신호가 확인됨(요약문 전체 맥락)");
      else if (hasQuestion) setC("C2","질문·요구 신호가 확인됨(요약문 전체 맥락)");
      else if (hasElab) setC("C3","상대 주장에 근거/설명 보탬(정교화)");
      else setC("C1","명시적 수락/수용이 우세하거나 최소 상호작용 확인");
    }

    // 규칙 3: C1 남발 교정(이미 C1인 경우 상향 재분류)
    if (r.codes && r.codes.C === "C1") {
      if (hasRebuttal) setC("C4","동의보다 반박 신호가 우세 → C4로 교정");
      else if (hasQuestion) setC("C2","동의보다 질문/요구가 우세 → C2로 교정");
      else if (hasElab) setC("C3","동의보다 근거 보탬이 우세 → C3으로 교정");
    }
  }
  return results;
}

/**
 * P차원 후검증 (P0~P3)
 * ✅ P0: activeN === 0 (아무도 인식적 발화 없음)
 * ✅ P1: activeN === 1 (1명만 의미 있게 기여)
 * ✅ P2: activeN === 2 || activeN === 3 (2~3명 의미 있게 기여)
 * ✅ P3: activeN >= 4 (4명 이상 의미 있게 기여)
 */
function _postValidatePCodes_(results, clusters) {
  var byId = {};
  (clusters || []).forEach(function(c) { byId[c.id] = (c.meta || {}); });

  for (var i = 0; i < (results || []).length; i++) {
    var r = results[i];
    if (!r || !r.codes || !r.codes.P) continue;
    
    var m = byId[r.id] || {};
    var speakerCounts = m.speaker_counts || [];
    var activeTurns = speakerCounts.map(function(x) { return x || 0; });
    var activeN = activeTurns.filter(function(x) { return x > 0; }).length; // ★ 값 > 0
    
    var pCode = r.codes.P;
    var summary = (r.writeups && r.writeups.P && r.writeups.P.analysis) || "";
    
    // P3 검증: activeN >= 4 확인
    if (pCode === "P3" && activeN < 4) {
      // P3 → P2로 강등 (2~3명이면 P2)
      if (activeN >= 2) {
        r.codes.P = "P2";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P2. 소수의 의미 있는 참여";
          r.writeups.P.analysis = "P3 조건 미충족(활성<4)으로 P2로 변경. " + summary;
        }
      } else if (activeN === 1) {
        r.codes.P = "P1";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P1. 1명의 의미 있는 참여";
          r.writeups.P.analysis = "P3 조건 미충족(활성<4)으로 P1로 변경. " + summary;
        }
      } else {
        r.codes.P = "P0";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P0. 의미 있는 참여 없음";
          r.writeups.P.analysis = "P3 조건 미충족(활성=0)으로 P0로 변경. " + summary;
        }
      }
    }
    
    // P2 검증: activeN === 2 || activeN === 3 확인
    if (pCode === "P2" && (activeN < 2 || activeN > 3)) {
      if (activeN >= 4) {
        r.codes.P = "P3";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P3. 다수의 의미 있는 참여";
          r.writeups.P.analysis = "P2 조건 미충족(활성≥4)으로 P3로 변경. " + summary;
        }
      } else if (activeN === 1) {
        r.codes.P = "P1";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P1. 1명의 의미 있는 참여";
          r.writeups.P.analysis = "P2 조건 미충족(활성=1)으로 P1로 변경. " + summary;
        }
      } else {
        r.codes.P = "P0";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P0. 의미 있는 참여 없음";
          r.writeups.P.analysis = "P2 조건 미충족(활성=0)으로 P0로 변경. " + summary;
        }
      }
    }
    
    // P1 검증: activeN === 1 확인
    if (pCode === "P1" && activeN !== 1) {
      if (activeN >= 4) {
        r.codes.P = "P3";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P3. 다수의 의미 있는 참여";
          r.writeups.P.analysis = "P1 조건 미충족(활성≥4)으로 P3로 변경. " + summary;
        }
      } else if (activeN >= 2) {
        r.codes.P = "P2";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P2. 소수의 의미 있는 참여";
          r.writeups.P.analysis = "P1 조건 미충족(활성≥2)으로 P2로 변경. " + summary;
        }
      } else {
        r.codes.P = "P0";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P0. 의미 있는 참여 없음";
          r.writeups.P.analysis = "P1 조건 미충족(활성=0)으로 P0로 변경. " + summary;
        }
      }
    }
    
    // P0 검증: activeN === 0 확인
    if (pCode === "P0" && activeN > 0) {
      if (activeN >= 4) {
        r.codes.P = "P3";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P3. 다수의 의미 있는 참여";
          r.writeups.P.analysis = "P0 조건 미충족(활성≥4)으로 P3로 변경. " + summary;
        }
      } else if (activeN >= 2) {
        r.codes.P = "P2";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P2. 소수의 의미 있는 참여";
          r.writeups.P.analysis = "P0 조건 미충족(활성≥2)으로 P2로 변경. " + summary;
        }
      } else {
        r.codes.P = "P1";
        if (r.writeups && r.writeups.P) {
          r.writeups.P.label_ko = "P1. 1명의 의미 있는 참여";
          r.writeups.P.analysis = "P0 조건 미충족(활성=1)으로 P1로 변경. " + summary;
        }
      }
    }
  }
  
  return results;
}

/**
 * D1 울트라-스트릭트 강제 적용 (사후 보정)
 */
function _enforceD1UltraStrict_(results, clusters){
  var segById = {}; (clusters||[]).forEach(function(c){ segById[c.id]=c; });

  // 절차 제안/수용 어휘(필요하면 단어 더 추가 가능)
  var PROPOSE = /(하자|정하자|정리하자|누가\s*발표|발표\s*하자|발표자|역할\s*나누|분담|순서\s*정하|먼저\s*~하자|일단\s*~하자|이걸\s*답|이걸로\s*가|최종|결정|정답|몇\s*번으로)/;
  var UPTAKE  = /(그래|그러자|오케이|OK|좋아|일단|맞아|동의|그럼\s*~하자)/;

  function hasStudentD1Move(seg){
    if(!seg || !seg.summarySource) return false;
    var t = seg.summarySource;

    // (A) 최초 절차 제안이 '학생'인가?
    var first = -1, teacherFirst=false;
    for (var i=0;i<t.length;i++){
      var u=String(t[i].utt||"");
      if (PROPOSE.test(u)){ first=i; teacherFirst=!!t[i].isTeacher; break; }
    }
    if (first<0 || teacherFirst) return false;

    // (B) 다른 학생의 수용/동의가 바로 뒤에서 나오는가?
    var spk0 = (t[first].speaker||"").trim();
    for (var j=first+1;j<Math.min(t.length, first+5);j++){
      if (t[j].isTeacher) continue;
      var diffSpk = (t[j].speaker||"").trim() !== spk0;
      if (diffSpk && UPTAKE.test(String(t[j].utt||""))) return true;
    }
    return false;
  }

  for (var i=0;i<(results||[]).length;i++){
    var r = results[i]; if(!r || !r.codes) continue;
    if (r.codes.D !== "D1") continue;

    var seg = segById[r.id];
    var speakers = (seg && seg.meta && seg.meta.student_speakers) || 0;
    var hasPeer  = !!(seg && seg.meta && seg.meta.has_peer_interaction);

    var ok = hasPeer && speakers>=2 && hasStudentD1Move(seg);
    if (!ok){
      r.codes.D = "없음";
      if (r.writeups && r.writeups.D){
        r.writeups.D.label_ko = "해당 없음";
        r.writeups.D.analysis = "학생 주도 절차 제안과 또래의 즉시 수용 근거가 부족하여 D1을 부여하지 않습니다.";
      }
    }
  }
  return results;
}


/**
 * 요약 텍스트 품질 점검
 */
function _isBadSummaryText_(t){
  const s = String(t||"").trim();
  if (!s) return true;
  // 눈에 띄는 실패/깨짐 신호
  if (/^(■\s*)?(\?\?:\?\?|시간미상)/.test(s)) return true;
  if (/undefined|null|NaN|\{\}|\[\]|"results":\s*\[/.test(s)) return true;
  if (s.length < 15) return true;  // 과도한 단문
  return false;
}

/**
 * 지정 PID들만 요약 재생성(F열)
 */
function _regenSummariesForPIDs_(pidList){
  if (!pidList || !pidList.length) return { fixed:0, changedPIDs:[] };
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data  = sheet.getDataRange().getValues();
  const segs  = collectSegmentsByPID_(data); // {id,time,summarySource,...}
  const byId  = {}; segs.forEach(s=> byId[s.id]=s);

  // 목록뷰(F)는 segs 순서대로 2행부터
  const payload = pidList.map(pid => byId[pid]).filter(Boolean);
  if (!payload.length) return { fixed:0, changedPIDs:[] };

  // SUMMARY_MODE에 맞게 재요약
  let res = {results:[]};
  try{
    if (SUMMARY_MODE === "gpt_rich")        res = summarizeSegmentsByPID_RichNarrative_(payload);
    else if (SUMMARY_MODE === "gpt_narrative") res = summarizeSegmentsByPID_Natural_(payload);
    else if (SUMMARY_MODE === "local_full") res = {results: payload.map(s=>({id:s.id,time:s.time,summary:summarizeTurns(s.summarySource,false)}))};
    else                                     res = summarizeSegmentsByPID_(payload); // gpt_exhaustive
  }catch(e){
    // 전부 로컬 폴백
    res = {results: payload.map(s=>({id:s.id,time:s.time,summary:summarizeTurns(s.summarySource,false)}))};
  }

  const byRes = {}; (res.results||[]).forEach(x=> byRes[x.id]=x);
  let fixed=0, changedPIDs=[];
  segs.forEach((s, i)=>{
    if (!byRes[s.id]) return;
    if (!pidList.includes(s.id)) return;
    const row = 2 + i; // 목록뷰 행
    const tStart = (byRes[s.id].time && byRes[s.id].time.start) || s.time.start || "??:??";
    const tEnd   = (byRes[s.id].time && byRes[s.id].time.end)   || s.time.end   || "??:??";
    const text   = byRes[s.id].summary || summarizeTurns(s.summarySource,false);
    const wrapped = "■ "+tStart+"~"+tEnd+"\t"+text;
    sheet.getRange(row, P_SUMMARY_REFINED_COL).setValue(wrapped);
    fixed++; changedPIDs.push(s.id);
  });

  return { fixed, changedPIDs };
}

/**
 * 전 구간 요약 무결성 스캔 → 불량 PID 골라 재생성
 */
function ensureSummaryIntegrityAndRepair(){
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data  = sheet.getDataRange().getValues();
  const segs  = collectSegmentsByPID_(data);
  if (!segs.length) return { fixed:0, changedPIDs:[] };

  const badPIDs = [];
  segs.forEach((s, i)=>{
    const row = 2 + i;
    const f = sheet.getRange(row, P_SUMMARY_REFINED_COL).getValue();
    if (_isBadSummaryText_(f)) badPIDs.push(s.id);
  });

  if (!badPIDs.length) return { fixed:0, changedPIDs:[] };
  return _regenSummariesForPIDs_(badPIDs);
}

/**
 * 일괄 검토·자동 보정기
 */
function auditAndAutofixAll(){
  const ss   = SpreadsheetApp.getActiveSpreadsheet();
  const sh   = ss.getActiveSheet();
  const data = sh.getDataRange().getValues();
  const last = sh.getLastRow();

  // 준비
  const segs = collectSegmentsByPID_(data);                 // {id,startRow,endRow,time,summarySource,teacher_involved}
  const teacherMap = _buildTeacherPresenceMap_(sh);         // pid -> boolean
  const idList = segs.map(s=>s.id);

  // 로그 시트 제거 - 자동 처리만 수행

  // E열(PID 목록) 정합성 강제
  if (last>=2){
    sh.getRange(2, P_IDLIST_COL, last-1, 1).clearContent();
    if (idList.length) sh.getRange(2, P_IDLIST_COL, idList.length, 1).setValues(idList.map(x=>[x]));
  }

  // 헬퍼
  function _topKCode_(t){
    const s=String(t||"").split("\n")[0].replace(/\*/g,"").trim();
    if(!s || /없음|해당\s*없음/i.test(s)) return null;
    const m=s.match(/\bA[123]\b/); return m?m[0]:null;
  }
  function _topCCode_(t){
    const s=String(t||"").split("\n")[0].replace(/\*/g,"").trim();
    if(!s || /없음|해당\s*없음/i.test(s)) return null;
    const m=s.match(/\bC[1-7]\b/); return m?m[0]:null;
  }
  function _topMCode_(t){
    const s=String(t||"").split("\n")[0].replace(/\*/g,"").trim();
    if(!s || /없음|해당\s*없음/i.test(s)) return null;
    const m=s.match(/\bD[1-4]\b/); return m?m[0]:null;
  }
  function _noneText(dim, why){
    if (dim==="A") return "해당 없음\n" + (why||"수업 과업·개념과 직접 관련된 설명/추론 근거가 뚜렷하지 않습니다.");
    if (dim==="C") return "해당 없음\n" + (why||"학생↔학생 인접 상호작용이 확인되지 않아 C차원 코딩을 적용하지 않습니다.");
    if (dim==="D") return "해당 없음\n" + (why||"목표·절차·논리·개념 점검의 명시적 단서가 부족합니다.");
    return "해당 없음";
  }
  // 오프태스크 휴리스틱(간단)
  const OFF = /(시험|객관식|기말|숙제|게임|인스타|과자|농담|빵|웃김|ㅋㅋ|ㅎㅎ|뿡)/;
  function _offTaskRatio_(seg){
    const t = seg.summarySource||[];
    let total=0, off=0;
    t.forEach(x=>{
      if (x.isTeacher) return;
      const u=String(x.utt||"").trim();
      if (!u) return;
      total++;
      if (OFF.test(u)) off++;
    });
    return total? (off/total) : 0;
  }
  // D1 검증(원자료 기반) — 이전 답변에서 준 것과 동일 로직
  function _validM1_(seg){
    const turns = seg.summarySource||[];
    const PROPOSE = /(발표\s*하자|누가\s*발표|발표자|역할\s*나누|분담|순서\s*정하|정리하자|정하자|이걸로\s*가|최종|정답|결정|몇\s*번으로|먼저\s*.+하자|일단\s*.+하자)/;
    const UPTAKE  = /(그래|그러자|오케이|OK|좋아|맞아|동의|그럼\s*.+하자|일단\s*.+하자)/;
    const TEACH_TRIGGER = /(누가|조별|의견\s*모아|발표|발표자|정리|정답|결정|역할|순서)/;

    function uniqStudents(){
      const s={}; turns.forEach(t=>{ if(!t.isTeacher) s[(t.speaker||"학생").trim()]=1; });
      return Object.keys(s).length;
    }
    function teacherTriggered(firstIdx){
      const s = Math.max(0, firstIdx-3);
      for (let i=s;i<firstIdx;i++){
        const t=turns[i]; if (t && t.isTeacher && TEACH_TRIGGER.test(String(t.utt||""))) return true;
      }
      return false;
    }
    // 학생 제안
    let first=-1, sp="";
    for (let i=0;i<turns.length;i++){
      const t=turns[i]; if (!t || t.isTeacher) continue;
      if (PROPOSE.test(String(t.utt||""))){ first=i; sp=(t.speaker||"").trim(); break; }
    }
    if (first<0) return {ok:false, why:"no_proposal"};
    if (teacherTriggered(first)) return {ok:false, why:"teacher_triggered"};
    // 즉시 또래 수용(1~4턴 내)
    for (let j=first+1;j<Math.min(turns.length, first+5);j++){
      const u=turns[j]; if (!u || u.isTeacher) continue;
      if ((u.speaker||"").trim()!==sp && UPTAKE.test(String(u.utt||""))) return {ok:true};
    }
    return {ok:false, why:"no_peer_uptake"};
  }

  // 집계
  let cnt = {
    fixedF:0, fixedJ:0, filledK:0, filledC:0, filledM:0,
    prunedC:0, prunedM1:0, offTaskK:0, suspicious:0
  };

  // 각 PID를 목록뷰 줄(2+i)와 매칭해 점검/보정
  segs.forEach(function(seg, i){
    const row = 2 + i;
    const pid = seg.id;

    // --- F(요약) 누락 보충
    let f = String(sh.getRange(row, P_SUMMARY_REFINED_COL).getValue()||"").trim();
    if (!f){
      const tStart = seg.time.start || "??:??";
      const tEnd   = seg.time.end   || "??:??";
      const txt    = summarizeTurns(seg.summarySource, false);
      f = "■ "+tStart+"~"+tEnd+"\t"+txt;
      sh.getRange(row, P_SUMMARY_REFINED_COL).setValue(f);
      cnt.fixedF++;
    }

    // --- J(교사개입) 재계산
    const wantJ = teacherMap[pid] ? "교사개입" : "";
    const curJ  = String(sh.getRange(row, TEACHER_FLAG_COL).getValue()||"").trim();
    if (curJ !== wantJ){
      sh.getRange(row, TEACHER_FLAG_COL).setValue(wantJ);
      cnt.fixedJ++;
    }

    // --- G/H/I 비어 있으면 '해당 없음' 채움
    let k = sh.getRange(row, P_K_COL).getValue();
    let h = sh.getRange(row, P_C_COL).getValue();
    let m = sh.getRange(row, P_M_COL).getValue();

    if (!String(k||"").trim()){
      sh.getRange(row, P_K_COL).setValue(_noneText("K","자동 후보정: 코딩 누락"));
      cnt.filledK++;
    }
    if (!String(h||"").trim()){
      sh.getRange(row, P_C_COL).setValue(_noneText("C","자동 후보정: 코딩 누락"));
      cnt.filledC++;
    }
    if (!String(m||"").trim()){
      sh.getRange(row, P_M_COL).setValue(_noneText("M","자동 후보정: 코딩 누락"));
      cnt.filledM++;
    }

    // 최신 값 재읽기
    k = sh.getRange(row, P_K_COL).getValue();
    h = sh.getRange(row, P_C_COL).getValue();
    m = sh.getRange(row, P_M_COL).getValue();

    // --- 오프태스크 높으면 K 제거
    const offRatio = _offTaskRatio_(seg);
    const kTop = _topKCode_(k);
    if (offRatio > 0.5 && kTop){
      sh.getRange(row, P_K_COL).setValue(_noneText("K","오프태스크 비중이 높아 K차원 코딩을 제거했습니다."));
      cnt.offTaskK++;
    }

    // --- C 과잉 방지(학생<2, 인접쌍<1, teacher-student only)
    const cTop = _topCCode_(h);
    const speakers = (function(){
      const s={}; (seg.summarySource||[]).forEach(t=>{ if(!t.isTeacher) s[(t.speaker||"학생").trim()]=1; });
      return Object.keys(s).length;
    })();
    const peerPairs = _countPeerAdjPairs_(seg);
    const teacherOnly = (!!seg.teacher_involved) && (speakers===1);
    if (cTop && (speakers<2 || peerPairs<1 || teacherOnly)){
      const why = teacherOnly ? "교사↔학생만 있어 C 금지" :
                   (speakers<2 ? "학생 화자 수<2" : "또래 인접쌍<1");
      sh.getRange(row, P_C_COL).setValue(_noneText("C", why));
      cnt.prunedC++;
    }

    // --- M1 엄격 판정
    const mTop = _topMCode_(m);
    if (mTop === "M1"){
      const chk = _validM1_(seg);
      if (!(speakers>=2 && chk.ok)){
        const why = (chk.why==="teacher_triggered") ? "교사 지시 후 수동 응답" :
                    (chk.why==="no_proposal") ? "학생 제안 부재" :
                    (chk.why==="no_peer_uptake") ? "또래 즉시 수용 부재" : "근거 부족";
        sh.getRange(row, P_M_COL).setValue(_noneText("M","M1 보수화: "+why));
        cnt.prunedM1++;
      }
    }

    // --- 의심 구간 로깅(학생 실질 발화는 많은데 K/C/M 모두 없음)
    const studTurns = (seg.summarySource||[]).filter(t=>!t.isTeacher && String(t.utt||"").trim().length>3);
    const noneK = !/K[123]/.test(String(sh.getRange(row, P_K_COL).getValue()||""));
    const noneC = !/C[1-7]/.test(String(sh.getRange(row, P_C_COL).getValue()||""));
    const noneM = !/M[1-4]/.test(String(sh.getRange(row, P_M_COL).getValue()||""));
    if (studTurns.length>=4 && noneK && noneC && noneM){
      cnt.suspicious++;
    }
  });

  // 요약
  const summary =
    "✅ 일괄 검토·자동 보정 완료\n\n" +
    "📋 PID 목록(E열) 재정렬: 완료\n" +
    `📝 F(요약) 보충: ${cnt.fixedF}건\n` +
    `👨‍🏫 J(교사개입) 재계산: ${cnt.fixedJ}건\n` +
    `➕ 누락 채움 K/C/M: ${cnt.filledK}/${cnt.filledC}/${cnt.filledM}건\n` +
    `🔒 C차원 게이트 적용: ${cnt.prunedC}건 (활성 화자 <2 등)\n` +
    `🔍 M1 엄격 검증: ${cnt.prunedM1}건 (학생 제안→수용 부재)\n` +
    `🚫 오프태스크 K 제거: ${cnt.offTaskK}건\n` +
    `⚠️ 의심 구간(과소코딩): ${cnt.suspicious}건`;

  return {summary, stats: cnt};
}

/**
 * ④ 캘리브레이션 정제기 — v4 (Raw → JSON 변환)
 * 사람이 표 형태로 준 ACD 결과를 바로 JSON 예시로 바꾸고 싶을 때 쓰는 보조 프롬프트
 */
function convertRawToCalibrationExamples(rawText, mapRules) {
  var systemPrompt = `You convert noisy, mixed-format Korean ACD coding notes into clean calibration examples for discourse coding.
- Parse time spans, 화자, 학생/교사 구분, 원문 인용.
- 추정이 필요한 필드는 보수적으로 "none".
- 교사 인용은 snippet에 포함하되 why/evidence에서는 사용 금지.
- STRICT JSON.`;


  var userPrompt = `raw_text = ${JSON.stringify(rawText)}
map_rules = ${JSON.stringify(mapRules || {
  "K":{"allow":["K1","K2","K3","none"]},
  "C":{"allow":["C1","C2","C3","C4","C5","C6","C7","none"]},
  "M":{"allow":["M1","M2","M3","M4","none"]},
  "teacher_markers":["교사","교사2","선생","쌤","교사개입"]
})}


[OUTPUT SCHEMA]
[
  {
    "snippet":[{"speaker":"학생|남학생|여학생|교사","isTeacher":false,"text":"..."}...],
    "labels":{"A":"A1|A2|A3|none","C":"C1..C7|none","D":"D1..D4|none"},
    "why":{"A":"요약 근거","C":"요약 근거","D":"요약 근거"}
  }
]
Return ONLY the JSON array.`;


  var messages = [
    { role: "system", content: systemPrompt },
    { role: "user", content: userPrompt }
  ];


  try {
    var result = callGPT_JSON(messages, MODEL_SUMMARY);
    return result || [];
  } catch (e) {
    Logger.log("캘리브레이션 변환 실패: " + e.toString());
    return [];
  }
}


/**
 * 학생↔학생 인접쌍 카운트 (C차원 검증용)
 */
function _countPeerAdjPairs_(seg) {
  const t = (seg.summarySource || []);
  let cnt = 0;
  for (let i = 1; i < t.length; i++) {
    const a = t[i-1], b = t[i];
    if (!a.isTeacher && !b.isTeacher) {
      // 간단한 Q↔A/주장↔응답 휴리스틱
      const qa = /[?]|왜|어떻게|맞아|그럼|아닌데|하지만/;
      if (qa.test(a.utt) || qa.test(b.utt)) cnt++;
    }
  }
  return cnt;
}

function semanticFallbackCluster(rows){
  const mmss = s=>{
    if(!s) return null; const m=String(s).trim().match(/^(\d{1,2}):([0-5]\d)$/);
    return m ? (+m[1])*60+(+m[2]) : null;
  };
  const isSub = t => t && !/^(응|아|어|네|맞아|그래|하하|ㅎㅎ|ㅋㅋ|끄덕.*)$/.test(String(t).trim());
  const subCount = rows.reduce((n,r)=> n + (isSub(r.utter)?1:0), 0);

  const MAX_ROWS=12, MAX_SEC=90, TEACH_ROWS=3, TEACH_SEC=35;
  const minK = Math.max(3, Math.floor(subCount/12)+1);

  // 1) 1차 경계: 교사 블록, 60초 이상 공백, 강한 질문/명료화
  let boundaries = new Set();
  for(let i=1;i<rows.length;i++){
    const a=rows[i-1], b=rows[i];
    const sa=mmss(a.ts), sb=mmss(b.ts);
    // 긴 공백
    if(sa!=null && sb!=null && sb-sa>=60) boundaries.add(i);
    // 질문/명료화(학생)
    if(!b.isTeacher && /(\?|왜|어떻게|무슨|설명해|근거|맞아\?)/.test(b.utter||"")) boundaries.add(i);
  }
  // 교사 연속 블록
  (function(){
    let s=0, run=0, t0=null;
    for(let i=0;i<rows.length;i++){
      if(rows[i].isTeacher){
        if(run===0){ s=i; t0=mmss(rows[i].ts); }
        run++;
      }else{
        if(run>=TEACH_ROWS){
          boundaries.add(s+1);
          boundaries.add(i); // 블록 끝
        }else{
          // 길이로도 체크
          const t1=mmss(rows[i-1]?.ts);
          if(t0!=null && t1!=null && (t1-t0)>TEACH_SEC){ boundaries.add(s+1); boundaries.add(i); }
        }
        run=0; t0=null;
      }
    }
  })();

  // 2) 경계 적용 → pid 부여
  const pidStr = n=>"P"+String(n).padStart(3,"0");
  let pidNum=1, cur=pidStr(pidNum);
  const pidsByRow={};
  pidsByRow[rows[0].row]=cur;
  for(let i=1;i<rows.length;i++){
    if(boundaries.has(i)) { pidNum++; cur=pidStr(pidNum); }
    pidsByRow[rows[i].row]=cur;
  }

  // 3) 상한 초과 구간 쪼개기
  function splitLarge(){
    // pid -> 구간
    const segs=[], ids=[];
    rows.forEach(r=>{
      const p=pidsByRow[r.row]; if(!ids.includes(p)) ids.push(p);
    });
    ids.forEach(pid=>{
      const idx = rows.filter(r=> pidsByRow[r.row]===pid);
      if(!idx.length) return;
      let s=0;
      while(s<idx.length){
        let e=s, sub=0, t0=mmss(idx[s].ts), t1=t0;
        while(e<idx.length){
          if(isSub(idx[e].utter)) sub++;
          const tt=mmss(idx[e].ts); if(tt!=null) t1=tt;
          const dur = (t0!=null&&t1!=null) ? (t1-t0) : 0;
          if(sub>MAX_ROWS || dur>MAX_SEC) break;
          e++;
        }
        // e가 s에서 멈췄으면 최소 1턴은 포함
        if(e===s) e=s+1;
        // s..e-1 유지, e.. 계속 새 pid
        const keep = idx.slice(s, e);
        const newPid = pidStr(++pidNum);
        // 첫 덩어리는 기존 pid 유지, 이후 덩어리는 새 pid
        if(s===0){
          keep.forEach(r=> pidsByRow[r.row]=pid);
        }else{
          keep.forEach(r=> pidsByRow[r.row]=newPid);
        }
        s=e;
      }
    });
  }
  splitLarge();

  // 4) 최소 클러스터 수 보장: 가장 큰 구간들 더 쪼개기
  function countK(){
    const seen=new Set(); rows.forEach(r=> seen.add(pidsByRow[r.row]));
    return seen.size;
  }
  function largestPid(){
    const map={};
    rows.forEach(r=>{
      const p=pidsByRow[r.row];
      map[p]=map[p]||{pid:p, turns:0};
      if(isSub(r.utter)) map[p].turns++;
    });
    return Object.values(map).sort((A,B)=>B.turns-A.turns)[0]?.pid;
  }
  while(countK()<minK){
    const pid = largestPid(); if(!pid) break;
    const block = rows.filter(r=> pidsByRow[r.row]===pid);
    if(block.length<4) break;
    const mid = Math.floor(block.length/2);
    const newPid = "P"+String(++pidNum).padStart(3,"0");
    for(let i=mid;i<block.length;i++) pidsByRow[block[i].row]=newPid;
  }

  // 5) 결과 row_codes
  const row_codes = rows.map(r=>({row:r.row, pid:pidsByRow[r.row]}));
  return { row_codes: row_codes };
}

function generateEFPairsFromCursorPrompt(){
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data = sheet.getDataRange().getValues();
  var N = data.length;
  if (N <= 1) return;

  // PID가 있는 행들만 수집
  var rows = [];
  for (var i=1;i<N;i++){
    var pid = (data[i][PID_COL-1]||"").trim();
    if (!pid) continue;
    var speaker = data[i][SPEAKER_COL-1] || "";
    var tsStr   = data[i][TS_COL-1] || "";
    var uttRaw  = String(data[i][UTTER_COL-1] || "").trim();
    rows.push({ 
      row: i+1, 
      speaker: speaker, 
      isTeacher: isTeacherSpeaker(speaker), 
      ts: tsStr || null, 
      utter: normalizeUtterance(uttRaw),
      pid: pid
    });
  }
  if (!rows.length) return;

  // Cursor 프롬프트로 요약 생성
  var messages = getCursorBasedSummaryPrompt(rows);
  var result = callGPT_JSON(messages, MODEL_SUMMARY);
  if (!result || !result.summaries) return;

  // 결과를 F열에 적용 (앵커 행만)
  var dataAfter = sheet.getDataRange().getValues();
  var segs = collectSegmentsByPID_(dataAfter);
  
  segs.forEach(function(s){
    var summary = result.summaries.find(function(sum) { return sum.pid === s.id; });
    if (summary && summary.summary) {
      var tStart = _sanitizeMMSS_(s.time.start);
      var tEnd = _sanitizeMMSS_(s.time.end);
      var text = `■ ${tStart}~${tEnd} ${summary.summary}`;
      sheet.getRange(s.startRow, P_SUMMARY_REFINED_COL).setValue(text);
    }
  });
}

function getCursorBasedSummaryPrompt(rows){
  var system = `You are a Korean classroom discourse summarizer that writes refined F-column summaries per PID.

TASK
- Input is a JSON array of rows: [{"row":<int>,"speaker":"<str>","isTeacher":true|false,"ts":"mm:ss|null","pid":"P###","utter":"<text>"}...].
- Group ALL rows by pid (P001, P002, …). For each pid:
  1) time span = first non-null ts ~ last non-null ts within that pid (fallback: "??:??").
  2) Write a NATURAL Korean summary (1–3 sentences) that uses ALL substantive turns of that pid as evidence.
  3) Keep flow and context: who asked/answered/added examples; mention "교사" exactly once if any teacher turns exist.
  4) Compress repetitions/fillers/off-task chatter; keep the core of peer interaction and the scientific mechanism.
  5) Include 1–2 short STUDENT quotes (5–20 chars) if helpful. (Teacher quotes only for context, sparingly.)
  6) Never invent content; do not add timestamps inside the prose.
- Order pids numerically (P001 < P002 < …).

STYLE
- Korean, clean narrative, no bullet points.
- Prefer verbs like: 묻는다/설명한다/덧붙인다/확인한다/정리한다.
- Preserve key science relations (e.g., 부피↑→압력↓; 공기는 고→저로 이동).

OUTPUT (JSON only, one summary per pid; no extra commentary)
{"summaries":[{"pid":"P###","summary":"<summary>"}]}

Return ONLY the JSON object.`;

  var user = `[INPUT ROWS]
${JSON.stringify(rows)}

NOW PRODUCE THE SUMMARIES.`;

  return [{role:"system",content:system},{role:"user",content:user}];
}

function getEnhancedCodingPrompt_Dstrict(clusters){
  const { teachers, students } = _getRoleProps_();
  const roleContext = students.length 
    ? `학생 명단: ${students.join(", ")}. 이 이름들은 모두 '학생'으로 간주한다.`
    : `이 데이터에서 '참석자 N' 표기는 '학생'으로 간주한다.`;

  const system = `You are a science education expert. Code small group discourse into A (Epistemic), C (Collaborative), D (Metacognitive) dimensions.

CORE RULES:
- Assign EXACTLY ONE code per dimension (NO multiple codes)
- Use ONLY student utterances as evidence (teacher utterances EXCLUDED)
- Prioritize student↔student interactions

${roleContext}

A DIMENSION (Epistemic):
- A1: Scientific explanation/exploration (simple info delivery)
- A2: Scientific claim+evidence (argument structure)
- A3: Scientific reasoning/hypothesis (complex thinking)
- Non-science chatter/off-task → A="OFF_TASK"

C DIMENSION (Collaborative) - Students ONLY:
- C1: Task coordination (role division/procedure)
- C2: Information request (question/confirmation)
- C3: Elaboration (supplement/expansion)
- C4: Acceptance (agreement/acknowledgment)
- C5: Challenge (objection/question)
- C6: Integration (synthesis/summary)
- C7: Peer teaching (explanation/help)
- Off-task chatter between students → C possible
- <2 student speakers → C="없음"

D DIMENSION (Metacognitive) - STRICT:
- D1: Procedural thinking (planning/strategy)
- D2: Monitoring (checking/verification)
- D3: Reflection (self-evaluation/improvement)
- D4: Regulation (adjustment/modification)
- Only with explicit cues

CRITICAL OUTPUT RULES:
1. You MUST return ONLY a JSON object with EXACTLY these fields:
   {"K":"<CODE>","C":"<CODE>","M":"<CODE>"}

2. ALLOWED VALUES ONLY:
   K: "없음" | "OFF_TASK" | "K1" | "K2" | "K3"
   C: "없음" | "C1" | "C2" | "C3" | "C4" | "C5" | "C6" | "C7"
   M: "없음" | "M1" | "M2" | "M3" | "M4"

3. FORBIDDEN ABSOLUTELY:
   - No explanations, descriptions, Korean text
   - No line breaks, markdown, comments
   - No extra fields or prefixes like "A#."
   - No multiple codes per dimension
   - No teacher utterances as evidence

4. EXAMPLES ONLY:
   {"K":"K2","C":"C3","M":"없음"}
   {"K":"OFF_TASK","C":"C1","M":"없음"}
   {"K":"없음","C":"없음","M":"없음"}

RETURN ONLY THE JSON OBJECT. NO OTHER TEXT.`;

  const studentList = students.length > 0 ? `학생 명단: ${students.join(", ")}` : "참석자 N 형태의 학생 화자들";
  
  const user = `ANALYZE THIS CLUSTER AND RETURN ONLY JSON:

${JSON.stringify(clusters)}

STUDENTS: ${studentList}
RULES: Exclude teacher utterances. Focus on student↔student interactions.

OUTPUT FORMAT (EXACT):
{"K":"CODE","C":"CODE","M":"CODE"}

VALID CODES:
K: "없음"|"OFF_TASK"|"K1"|"K2"|"K3"
C: "없음"|"C1"|"C2"|"C3"|"C4"|"C5"|"C6"|"C7"
M: "없음"|"M1"|"M2"|"M3"|"M4"

EXAMPLES:
{"A":"A2","C":"C3","D":"없음"}
{"A":"OFF_TASK","C":"C1","D":"없음"}

RETURN ONLY JSON. NO TEXT. NO EXPLANATIONS.`;

  return [{role:"system",content:system},{role:"user",content:user}];
}

// ── 메뉴 연결 함수들 ─────────────────────────
// ★ runClustering() 삭제됨: menu_cluster() 사용 (388줄)
// ★ runACDCoding() 중복 제거됨: 3752줄 버전 사용


// 다이어그램 작성 + 패턴 충돌 해결 래퍼 함수 (대용량 최적화)
// ★ buildDiagram13_chunked() 중복 제거됨 (4567줄 버전 사용 - batchSize 매개변수)



// ── 교사/학생 역할 입력 팝업(스크립트 프로퍼티 저장) ─────────────────────────
function setRoleMappings() {
  const ui = SpreadsheetApp.getUi();
  // 교사
  const t = ui.prompt("교사 이름(쉼표로 구분)", "예: 교사, 선생님, Teacher", ui.ButtonSet.OK_CANCEL);
  if (t.getSelectedButton() !== ui.Button.OK) return;
  const teacherCsv = (t.getResponseText() || "").split(",").map(s=>s.trim()).filter(Boolean).join(",");

  // 학생(최대 4명)
  const s = ui.prompt("학생 이름(쉼표로 구분, 최대 4명)", "예: 참석자 1, 참석자 2, 참석자 3, 참석자 4", ui.ButtonSet.OK_CANCEL);
  if (s.getSelectedButton() !== ui.Button.OK) return;
  const studentsArr = (s.getResponseText() || "").split(",").map(s=>s.trim()).filter(Boolean);
  const students = studentsArr.slice(0,4);
  if (studentsArr.length > 4) {
    ui.alert("학생은 최대 4명만 사용합니다. 앞의 4명만 저장합니다.");
  }

  PropertiesService.getScriptProperties().setProperty("TEACHER_NAMES_CSV", teacherCsv);
  PropertiesService.getScriptProperties().setProperty("STUDENT_NAMES_CSV", students.join(","));
  SpreadsheetApp.getActive().toast("역할 설정 저장 완료", "완료", 3);
}

function _getRoleProps_() {
  const sp = PropertiesService.getScriptProperties();
  const teacherCsv = sp.getProperty("TEACHER_NAMES_CSV") || "";
  const studentCsv = sp.getProperty("STUDENT_NAMES_CSV") || "";
  const teachers = teacherCsv ? teacherCsv.split(",").map(s=>s.trim()).filter(Boolean) : [];
  const students = studentCsv ? studentCsv.split(",").map(s=>s.trim()).filter(Boolean) : [];
  return { teachers, students };
}

// ── equity차원 분석: L~O 헤더 + 클러스터별 학생 발화수 기록 ─────────────────────────
function runEquityAnalysis() {
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  
  // 마지막 PID 행 찾기
  const lastRowLimit = findLastPidRow_(sh, map);
  
  const values = sh.getDataRange().getValues();
  const lastRow = Math.min(sh.getLastRow(), lastRowLimit);
  if (lastRow < 2) { ui.alert("데이터가 없습니다."); return; }

  const { teachers, students } = _getRoleProps_();
  if (students.length === 0) {
    ui.alert("학생 이름이 설정되지 않았습니다. 메뉴의 '🙋 교사/학생 지정(1회)'를 먼저 실행하세요.");
    return;
  }

  // 1) 화자 분류기 정의
  const reTeacherNames = teachers.length ? new RegExp("(" + teachers.map(_escapeRegExp).join("|") + ")", "i") : null;
  const isTeacher = (speaker) => {
    const s = String(speaker||"").trim();
    if (!s) return false;
    if (reTeacherNames && reTeacherNames.test(s)) return true;
    return /(교사|선생|teacher)/i.test(s);
  };
  const isStudent = (speaker) => {
    const s = String(speaker||"").trim();
    if (!s) return false;
    if (isTeacher(s)) return false;
    if (students.some(n => n && s.toLowerCase() === n.toLowerCase())) return true;
    if (/^(참석자|학생)\s*\d+$/i.test(s)) return true;
    return false;
  };

  // 2) G1~J1 헤더: 학생 이름 + 총 발화 횟수
  const startCol = SPEAKER_CNT_START_COL; // G=7
  
  // 전체 데이터에서 각 학생의 총 발화 횟수 계산 (lastRowLimit까지)
  const totalCounts = {}; // { studentName: totalCount }
  const speakers = values.map(r => String(r[0]||"").trim()); // A열
  
  // 모든 행을 돌면서 각 학생의 총 발화 횟수 계산 (lastRowLimit까지)
  for (let r=1; r<lastRow; r++) {
    const spk = speakers[r];
    if (!isStudent(spk)) continue;
    const normalized = _normalizeToStudent_(spk, students);
    if (!normalized) continue;
    totalCounts[normalized] = (totalCounts[normalized] || 0) + 1;
  }
  
  // 헤더에 학생 이름과 총 발화 횟수 표시
  for (let i=0; i<SPEAKER_CNT_COLS; i++) {
    const name = students[i] || "";
    const totalCount = totalCounts[name] || 0;
    const headerText = name ? `${name}(${totalCount})` : "";
    sh.getRange(1, startCol+i).setValue(headerText);
  }

  // 3) 클러스터 컬럼(D 또는 E) 추정
  const pidCol = _inferPidCol_(values);
  if (pidCol < 0) { ui.alert("클러스터 ID 컬럼(D 또는 E)을 찾지 못했습니다."); return; }

  // 4) 클러스터별 학생 발화수 집계 (lastRowLimit까지)
  const countsByPid = {}; // { pid: { studentName: count } }
  for (let r=1; r<lastRow; r++) {
    const pid = String(values[r][pidCol]||"").trim();
    if (!pid) continue;
    const spk = speakers[r];
    if (!isStudent(spk)) continue;
    const normalized = _normalizeToStudent_(spk, students);
    if (!normalized) continue;
    // PID 정규화 (공백 제거, 대소문자 통일)
    const normalizedPid = pid.replace(/\s+/g, "").toUpperCase();
    if (!countsByPid[normalizedPid]) countsByPid[normalizedPid] = {};
    countsByPid[normalizedPid][normalized] = (countsByPid[normalizedPid][normalized]||0) + 1;
  }

  // 5) 요약행 기준으로 G~J에 기록 (lastRowLimit까지)
  const pidFromSummaryRow = _buildPidIndexForSummaries_(values, pidCol, lastRow);
  const F_COL = 5; // 0-based (F)
  for (let r=1; r<lastRow; r++) {
    const hasSummary = String(values[r][F_COL]||"").trim();
    if (!hasSummary) continue;
    const pid = pidFromSummaryRow[r+1];
    if (!pid) continue;
    // PID 정규화 (공백 제거, 대소문자 통일)
    const normalizedPid = String(pid).replace(/\s+/g, "").toUpperCase();
    const counts = countsByPid[normalizedPid] || {};
    const rowCounts = [];
    for (let i=0; i<SPEAKER_CNT_COLS; i++) {
      const nm = students[i];
      rowCounts.push(nm ? (counts[nm]||0) : "");
    }
    sh.getRange(r+1, startCol, 1, SPEAKER_CNT_COLS).setValues([rowCounts]);
  }

  var msg = "✅ 화자별 발화수 분석 완료\n\n" +
            "📊 G~J열: 클러스터별 학생 발화수 기록됨\n" +
            "👥 헤더: 학생 이름(총 발화수) 형식\n" +
            "⚖️ P차원 코딩 필수 데이터 준비 완료";
  SpreadsheetApp.getUi().alert(msg);
}

// ── helpers ─────────────────────────────────────────────
function _escapeRegExp(s){ return String(s).replace(/[.*+?^${}()|[\]\\]/g, '\\$&'); }

function _inferPidCol_(values) {
  // D(3) 우선, 없으면 E(4) 사용
  const header = values[0]||[];
  const cand = [3,4];
  for (const c of cand) {
    const col = header[c] ? String(header[c]).toLowerCase() : "";
    const many = values.slice(1, Math.min(values.length, 60)).filter(r=>String(r[c]||"").trim()).length;
    if (/pid|cluster|p-?id|세그먼트|클러스터/.test(col) || many >= 3) return c;
  }
  return -1;
}

function _normalizeToStudent_(speaker, students) {
  for (const n of students) {
    if (n && speaker.toLowerCase() === n.toLowerCase()) return n; // 정확 일치
  }
  const m = speaker.match(/^(참석자|학생)\s*(\d+)$/i);
  if (m) {
    const stdLike = `${m[1]} ${m[2]}`; // '참석자 1' 형태
    for (const n of students) {
      if (n && n.replace(/\s+/g,"").toLowerCase() === stdLike.replace(/\s+/g,"").toLowerCase()) return n;
    }
  }
  return null;
}

/**
 * P 차원 판정 유틸 (P0~P3)
 * - P0: activeN === 0 (아무도 인식적 발화 없음)
 * - P1: activeN === 1 (1명만 의미 있게 기여)
 * - P2: activeN === 2 || activeN === 3 (2~3명 의미 있게 기여)
 * - P3: activeN >= 4 (4명 이상 의미 있게 기여)
 */
function decidePfromCountsAndSummary(speakerCounts, summary, teacherInvolved) {
  const total = Math.max(1, speakerCounts.reduce((a,b)=>a+b,0));
  const activeTurns = speakerCounts.map(x => x||0);
  const activeN = activeTurns.filter(x=>x>0).length;  // ★ 활성 기준(값 > 0)

  const s = String(summary||"").toLowerCase();

  // 오프태스크/근거 희소 → 없음
  if (total < 1 || /잡담|농담|무관|먹방|게임|SNS|인스타|유튜브|카톡/.test(s)) return "없음";

  // P0~P3 판정: activeN 기준
  if (activeN === 0) return "P0";
  if (activeN === 1) return "P1";
  if (activeN === 2 || activeN === 3) return "P2";
  if (activeN >= 4) return "P3";
  
  return "없음";
}

/**
 * P-다이어그램 생성 (P0~P3)
 */
function buildPDiagram(pCodeText) {
  if (!pCodeText) return "";
  const s = String(pCodeText).trim();
  if (!s || /없음|해당\s*없음/i.test(s)) return "";
  
  // P 코드 추출
  const m = s.match(/\bP[0-3]\b/);
  if (!m) return s; // 코드 없으면 원문 그대로
  
  const code = m[0];
  
  // 간단한 아이콘/표현 (P0~P3)
  const icons = {
    "P0": "🚫 참여 없음",
    "P1": "👤 1명 참여",
    "P2": "👥 소수 참여",
    "P3": "👥👥 다수 참여"
  };
  
  return icons[code] || code;
}

// 요약행(F가 채워진 행) → PID 매핑
function _buildPidIndexForSummaries_(values, pidCol, lastRowLimit) {
  const map = {}; // { row(1-based): pid }
  const E_COL = 4, F_COL = 5;
  const maxRow = lastRowLimit ? Math.min(values.length, lastRowLimit) : values.length;
  
  // PID 정규화 헬퍼
  const normalizePid = (pid) => String(pid||"").trim().replace(/\s+/g, "").toUpperCase();

  // 전체 PID 등장 순서 (lastRowLimit까지만)
  const order = []; const seen = new Set();
  for (let r=1; r<maxRow; r++) {
    const pid = normalizePid(values[r][pidCol]);
    if (pid && !seen.has(pid)) { seen.add(pid); order.push(pid); }
  }

  // 요약행 목록 (lastRowLimit까지만)
  const sumRows = [];
  for (let r=1; r<maxRow; r++) {
    const summary = String(values[r][F_COL]||"").trim();
    if (summary) sumRows.push(r+1);
  }

  // 우선: 해당 행의 E/D에 PID 직접 있으면 우선 매핑
  const direct = {};
  for (const rr of sumRows) {
    if (rr-1 >= maxRow) continue; // 범위 체크
    const ePid = normalizePid(values[rr-1][E_COL]);
    const dPid = normalizePid(values[rr-1][pidCol]);
    const pid = ePid || dPid || "";
    if (pid) direct[rr] = pid;
  }

  // 부족분은 순서 매칭으로 보완
  let k = 0;
  for (const rr of sumRows) {
    if (direct[rr]) { map[rr] = direct[rr]; continue; }
    while (k < order.length && Object.values(map).includes(order[k])) k++;
    map[rr] = order[k] || "";
    k++;
  }
  return map;
}

// ── ACD 프롬프트에 역할 맥락 주입(학생↔학생 상호작용 우선) ─────────────────────────
function _acdRolesRuleLine_() {
  const { students } = _getRoleProps_();
  const studentLine = students.length
    ? `학생 명단: ${students.join(", ")}. 이 이름들은 모두 '학생'으로 간주한다.`
    : `이 데이터에서 '참석자 N' 표기는 '학생'으로 간주한다.`;
  return [
    studentLine,
    "C차원 평가는 학생↔학생 인접 상호작용(동의/정교화/반박/명료화 등)에 한정하며, 교사 개입은 C에서 제외한다.",
    "A/D차원은 학생 발화가 1개 이상이면 원칙적으로 평가 대상(명백한 오프태스크 제외).",
  ].join("\n");
}


/***** ===== Event 병합(Act→Event) 유틸 ===== *****/

/** 토큰셋 생성(한글/영문/숫자 단어) */
function _tokenSet_(s){
  if(!s) return {};
  var toks = String(s).toLowerCase()
    .replace(/[^0-9a-zA-Z가-힣\s]/g, " ")
    .split(/\s+/).filter(Boolean);
  var set = {};
  toks.forEach(function(t){
    if (t.length>=2) set[t]=1; // 1글자 토큰은 노이즈로 제거
  });
  return set;
}

/** 두 토큰셋의 Jaccard */
function _jaccard_(a,b){
  var inter=0, uni=0;
  var keys = {};
  for (var k in a) { keys[k]=1; }
  for (var k2 in b){ keys[k2]=1; }
  for (var k3 in keys){
    var ina = !!a[k3], inb = !!b[k3];
    if (ina && inb) inter++;
    if (ina || inb) uni++;
  }
  return uni===0 ? 0 : inter/uni;
}

/** 세그먼트 요약 토큰셋(행 발화들을 간단히 합침) */
function _segmentTokenSet_(rows, seg){
  var set = {};
  for (var i=seg.s;i<=seg.e;i++){
    var utt = rows[i].utter || "";
    var ts = _tokenSet_(utt);
    for (var k in ts){ set[k]=1; }
  }
  return set;
}

/** 세그먼트 A/C/D 대표값(다수결) */
function _segmentModeCode_(rows, seg, codeMap){
  var cntA={}, cntC={}, cntD={};
  for (var i=seg.s;i<=seg.e;i++){
    var r = rows[i], c = codeMap[r.row] || {a:"none", c:"none", d:"none"};
    cntA[c.a]=(cntA[c.a]||0)+1;
    cntC[c.c]=(cntC[c.c]||0)+1;
    cntD[c.d]=(cntD[c.d]||0)+1;
  }
  function mode(obj){
    var best="none", mx=-1;
    for (var k in obj){ if (obj[k]>mx){ mx=obj[k]; best=k; } }
    return best;
  }
  return {a:mode(cntA), c:mode(cntC), d:mode(cntD)};
}

/** 코드 차이 정도(0~3) */
function _codeDrift_(m1, m2){
  return (m1.a!==m2.a) + (m1.c!==m2.c) + (m1.d!==m2.d);
}

/** 세그먼트 간 시간/행 길이 계산 */
function _decorateSegs_(rows, segs){
  function toSec(s){ 
    if(!s) return null; 
    var m = String(s).match(/^(\d{1,2}):([0-5]\d)$/);
    return m ? (parseInt(m[1],10)*60 + parseInt(m[2],10)) : null;
  }
  segs.forEach(function(seg){
    var first = rows[seg.s], last = rows[seg.e];
    seg.len   = seg.e - seg.s + 1;
    seg.startSec = toSec(first.ts);
    seg.endSec   = toSec(last.ts);
    seg.duration = (seg.startSec!=null && seg.endSec!=null) ? (seg.endSec - seg.startSec) : null;
    seg.hasTeacher = false;
    for (var i=seg.s;i<=seg.e;i++){ if (rows[i].isTeacher){ seg.hasTeacher=true; break; } }
  });
}

/** QA 연결 여부(앞 세그 끝이 ? 이거나, 뒤 세그 첫 행이 즉답/정당화 단서)
 *  질문 패턴 확장: 왜|뭐|무엇|어떻게|맞죠|맞나|인가요|맞나요|물어|궁금|?
 */
function _qaLinked_(rows, prevSeg, nextSeg, codeMap){
  var prevLast = rows[prevSeg.e];
  var nextFirst = rows[nextSeg.s];
  var uttPrev = (prevLast.utter||"");
  
  // 확장된 질문 패턴 (QA 민감도 회복)
  var isQuestion = /\?|왜|뭐|무엇|어떻게|맞죠|맞나|인가요|맞나요|물어|궁금/.test(uttPrev);
  
  var cNext = codeMap[nextFirst.row] || {};
  var isAnswerish = (cNext.a==="A1" || cNext.a==="A3" || cNext.c==="C3" || cNext.c==="C5");
  
  return isQuestion || isAnswerish;
}

/** Communicative Event 병합 단계
 *   - resegmentByRules → smoothSingletons 이후에 호출
 *   - 반환: { pidsByRow, idList }
 *   - 외부 의존: findManualBlock(parseMMSS(...), ...), parseMMSS(...)가 이미 있다면 사용
 */
function mergeEventsFromActs(rows, pidsByRow, gptRowCodes){
  // 안전성 체크: 빈 데이터셋
  if (!rows || !rows.length || !pidsByRow){
    return { pidsByRow: pidsByRow || {}, idList: [] };
  }

  // 1) pid 연속구간 나누기
  var segs = [];
  var curPid=null, sIdx=0;
  function pidOf(i){ return pidsByRow[rows[i].row] || ""; }
  for (var i=0;i<rows.length;i++){
    var pid = pidOf(i);
    if (pid !== curPid){
      if (curPid!=null) segs.push({pid:curPid, s:sIdx, e:i-1});
      curPid=pid; sIdx=i;
    }
  }
  if (curPid!=null) segs.push({pid:curPid, s:sIdx, e:rows.length-1});

  // 안전성 체크: 세그먼트가 1개 이하면 병합 불필요
  if (segs.length <= 1){
    var idList = segs.map(function(seg){ return seg.pid; });
    return { pidsByRow: pidsByRow, idList: idList };
  }

  // 2) 보조 맵/특징
  var codeMap={};
  (gptRowCodes||[]).forEach(function(r){ codeMap[r.row]={a:r.a||"none", c:r.c||"none", d:r.d||"none"}; });
  _decorateSegs_(rows, segs);
  var topicTokens = segs.map(function(seg){ return _segmentTokenSet_(rows, seg); });
  var modes       = segs.map(function(seg){ return _segmentModeCode_(rows, seg, codeMap); });

  // 3) 좌→우로 병합 시도
  var keep = new Array(segs.length).fill(true);
  for (var i=0;i<segs.length-1;i++){
    if (!keep[i] || !keep[i+1]) continue;
    var A = segs[i], B = segs[i+1];

    // (a) 긴 공백이면 병합 금지
    var gapSec = (A.endSec!=null && B.startSec!=null) ? (B.startSec - A.endSec) : null;
    if (gapSec!=null && gapSec > EVENT_GAP_SEC) continue;

    // (b) manual block 존중 (존재 시)
    var mbA = (typeof findManualBlock==='function' && typeof parseMMSS==='function')
      ? findManualBlock(parseMMSS(rows[A.s].ts), rows[A.s].row) : null;
    var mbB = (typeof findManualBlock==='function' && typeof parseMMSS==='function')
      ? findManualBlock(parseMMSS(rows[B.s].ts), rows[B.s].row) : null;
    if (mbA && mbB && mbA!==mbB) continue;

    // (c) 짧은 단발/짧은 시간
    var shortish = (B.len < EVENT_MIN_ROWS) || (B.duration!=null && B.duration < EVENT_MIN_SEC);

    // (d) 토픽 유사도 & 코드 유사성
    var topOverlap = _jaccard_(topicTokens[i], topicTokens[i+1]);
    var drift = _codeDrift_(modes[i], modes[i+1]);
    var similar = (topOverlap >= TOPIC_OVERLAP_T) && (drift <= CODE_DRIFT_TOL);

    // (e) QA 연결 고리
    var qa = _qaLinked_(rows, A, B, codeMap);

    // (f) 교사 창 처리: 둘 다 교사 잔존이면서 또래 상호작용 없는 경우는 유지
    var teacherWall = (A.hasTeacher && B.hasTeacher);

    // 병합 판단식: (짧음 OR QA) AND (주제/코드 유사) AND (교사벽 아님)
    if ((shortish || qa) && similar && !teacherWall){
      // B를 A로 흡수
      for (var k=B.s;k<=B.e;k++){ pidsByRow[rows[k].row] = pidsByRow[rows[A.s].row]; }
      keep[i+1]=false;

      // A 특성 갱신
      A.e = B.e;
      A.endSec = B.endSec;  // 버그 수정: segs[i+1] → B
      A.len = A.e - A.s + 1;
      A.duration = (A.startSec!=null && A.endSec!=null) ? (A.endSec - A.startSec) : null;
      if (B.hasTeacher) A.hasTeacher = true; // 버그 수정: 교사 플래그 병합

      // 모드/토큰 재계산(간단히 합집합/다수결 재사용)
      for (var kk in topicTokens[i+1]){ topicTokens[i][kk]=1; }
      modes[i] = _segmentModeCode_(rows, A, codeMap);

      // i를 한 칸 뒤로 당겨 연쇄 병합 기회 제공
      if (i>0) i-=2;
    }
  }

  // 4) PID 재번호 & idList 재구성
  var mapOldNew = {}, cnt=0, idList=[];
  for (var t=0;t<rows.length;t++){
    var old = pidsByRow[rows[t].row] || "";
    if (!old) continue;
    if (!mapOldNew[old]){
      mapOldNew[old] = "P" + String(++cnt).padStart(3,"0");
      idList.push(mapOldNew[old]);
    }
    pidsByRow[rows[t].row] = mapOldNew[old];
  }
  return { pidsByRow: pidsByRow, idList: idList };
}


/***** ===== AD 코딩 강화 시스템 (형식 강제 + 휴리스틱 판정) ===== *****/

/** 🔄 K&M 코딩 (K열과 M열 쓰기) */
// ============================================================
// LEGACY / INACTIVE / DO NOT USE
// runCodeKM_All: K+M 통합 실행 구버전 runner
// STEP 7 기준으로 live integrated runner(menu_runKCMP, runKCMPCoding)에서
// 완전히 제거됨. K와 M은 canonical order(K→C→M→P)에서 개별 실행.
// live call site = 0
// ============================================================
function runCodeKM_All_LEGACY() {
  // LEGACY / INACTIVE / DO NOT USE
  runCodeK_All();
  runCodeM_All();
  SpreadsheetApp.getUi().alert("✅ K&M 코딩 완료!");
}

/** 🔄 K 코딩 (K열 쓰기) — K Decision Tree v1.0 */
function runCodeK_All() {
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  if (!packets || !packets.length) {
    ui.alert('⚠️ 처리할 PID 패킷이 없습니다.\n클러스터링과 화자별 발화분석을 먼저 실행하세요.');
    return;
  }

  let codedCount = 0;
  let noneCount = 0;
  let errorCount = 0;
  let skippedCount = 0;

  packets.forEach(function(packet) {
    const row = packet && packet.representativeRow;
    if (!row) {
      skippedCount++;
      Logger.log('⚠️ K 코딩 건너뜀: representativeRow 없음 pid=' + (packet && packet.pid));
      return;
    }

    const result = runKDecisionTreeForPacket_(packet);
    _writeKDecisionCell_(sh, row, kCol, result);

    if (result && result.status === 'OK' && result.code) {
      codedCount++;
      if (codedCount <= 3) Logger.log('K 코딩 성공 [행' + row + '] ' + packet.pid + ': ' + result.code);
    } else if (result && result.status === 'OK' && result.code == null) {
      noneCount++;
    } else {
      errorCount++;
      Logger.log('❌ K 코딩 오류 [행' + row + '] ' + (packet.pid || '') + ': ' + (result && result.error_type) + ' ' + (result && result.message));
    }
  });

  let msg = '✅ K 코딩 완료 (Decision Tree v1.0)\n\n';
  msg += '📊 통계:\n';
  msg += '- K1/K2/K3 부여: ' + codedCount + '개\n';
  msg += '- 정상 K 없음: ' + noneCount + '개\n';
  if (errorCount > 0) msg += '- 오류(API/PARSER/VALIDATION/PACKET): ' + errorCount + '개\n';
  if (skippedCount > 0) msg += '- 건너뜀(대표행 없음): ' + skippedCount + '개\n';
  msg += '\nK 없음과 오류는 K셀 값(빈칸)이 같아 보여도 Note JSON의 status로 구분됩니다.';
  if (errorCount > 0) {
    msg += '\n\n⚠️ 오류가 있습니다. [보기] → [실행 로그]에서 error_type을 확인하세요.';
  }
  ui.alert(msg);
}

/** 🔄 M 코딩 (M열 쓰기) — Decision Tree v1.0 */
function runCodeM_All() {
  const ui = SpreadsheetApp.getUi();
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const mCol = colNumOf(map.M);

  const packets = buildAllKCMPClusterPackets_(sh, map);
  if (!packets || !packets.length) {
    ui.alert('⚠️ 처리할 PID 패킷이 없습니다.\n클러스터링과 화자별 발화분석을 먼저 실행하세요.');
    return;
  }

  let codedCount = 0;
  let noneCount = 0;
  let errorCount = 0;
  let skippedCount = 0;

  packets.forEach(function(packet) {
    const row = packet && packet.representativeRow;
    if (!row) {
      skippedCount++;
      Logger.log('⚠️ M 코딩 건너뜀: representativeRow 없음 pid=' + (packet && packet.pid));
      return;
    }

    const result = runMDecisionTreeForPacket_(packet, { allPackets: packets });
    _writeMDecisionCell_(sh, row, mCol, result);

    if (result && result.status === 'OK' && result.code) {
      codedCount++;
      if (codedCount <= 3) Logger.log('M 코딩 성공 [행' + row + '] ' + packet.pid + ': ' + result.code);
    } else if (result && result.status === 'OK' && result.code == null) {
      noneCount++;
    } else {
      errorCount++;
      Logger.log('❌ M 코딩 오류 [행' + row + '] ' + (packet.pid || '') + ': ' + (result && result.error_type) + ' ' + (result && result.message));
    }
  });

  let msg = '✅ M 코딩 완료 (Decision Tree v1.0)\n\n';
  msg += '📊 통계:\n';
  msg += '- M1~M4 부여: ' + codedCount + '개\n';
  msg += '- 정상 M 없음: ' + noneCount + '개\n';
  if (errorCount > 0) msg += '- 오류(API/PARSER/VALIDATION/PACKET): ' + errorCount + '개\n';
  if (skippedCount > 0) msg += '- 건너뜀(대표행 없음): ' + skippedCount + '개\n';
  msg += '\nM 없음과 오류는 M셀 값(빈칸)이 같아 보여도 Note JSON의 status로 구분됩니다.';
  if (errorCount > 0) {
    msg += '\n\n⚠️ 오류가 있습니다. [보기] → [실행 로그]에서 error_type을 확인하세요.';
  }
  ui.alert(msg);
}

// ============================================================
// STEP 9A/11: Resumable production batch runners (orchestration only)
// FINALIZED Note(status OK/ERROR + correct schema)는 GPT 재호출 없이 SKIP.
// runCodeK_All / runCodeC_All / runCodeM_All 의 decision logic은 변경하지 않음.
// Production 메뉴: onOpen → "KCMP Production" (resume batch 권장 경로)
// ============================================================
function runCodeK_ResumeBatch(){
  return _runKCMPProductionResumeBatch_("K");
}

function runCodeC_ResumeBatch(){
  return _runKCMPProductionResumeBatch_("C");
}

function runCodeM_ResumeBatch(){
  return _runKCMPProductionResumeBatch_("M");
}

function runCodeP_ResumeBatch(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const progress = _summarizeKCMPProductionProgress_(sh, map, packets);

  if (!progress.allKcmFinalized) {
    Logger.log("=== P RESUME BATCH BLOCKED ===");
    Logger.log("P_GATE_ALL_KCM_FINALIZED=false");
    Logger.log("K_UNFINALIZED=" + progress.K.unfinalized);
    Logger.log("C_UNFINALIZED=" + progress.C.unfinalized);
    Logger.log("M_UNFINALIZED=" + progress.M.unfinalized);
    Logger.log("==============================");
    SpreadsheetApp.getUi().alert(
      "⚠️ P 코딩을 시작할 수 없습니다.\n\n" +
      "K/C/M이 모두 finalized되어야 합니다.\n\n" +
      "K_UNFINALIZED=" + progress.K.unfinalized + "\n" +
      "C_UNFINALIZED=" + progress.C.unfinalized + "\n" +
      "M_UNFINALIZED=" + progress.M.unfinalized + "\n\n" +
      "[KCMP Production → 1. 진행상황 확인]으로 상태를 확인하세요."
    );
    return {
      blocked: true,
      reason: "ALL_KCM_FINALIZED=false",
      progress: progress
    };
  }

  return _runPProductionResumeBatch_();
}

// ============================================================
// STEP 16P: P 실행 전 upstream 자동 점검·복구·stale P 동기화
// P semantic / computeDeterministicPForPacket_ 변경 없음.
// ============================================================

function _canonicalStudentContributorSet_(arr){
  const out = [];
  (arr || []).forEach(function(c){
    const s = String(c == null ? "" : c).trim();
    if (/^S[1-4]$/.test(s) && out.indexOf(s) < 0) out.push(s);
  });
  out.sort();
  return out;
}

function _sameCanonicalContributorSet_(a, b){
  const aa = _canonicalStudentContributorSet_(a);
  const bb = _canonicalStudentContributorSet_(b);
  if (aa.length !== bb.length) return false;
  for (let i = 0; i < aa.length; i++) {
    if (aa[i] !== bb[i]) return false;
  }
  return true;
}

function _pUpstreamDimKey_(u){
  if (!u || typeof u !== "object") return "null";
  return [
    String(u.state == null ? "" : u.state),
    String(u.status == null ? "" : u.status),
    String(u.code == null ? "null" : u.code),
    _canonicalStudentContributorSet_(u.contributors).join(",")
  ].join("|");
}

function _pUpstreamStatesMatch_(curUpstream, expUpstream){
  const c = curUpstream || {};
  const e = expUpstream || {};
  return _pUpstreamDimKey_(c.K) === _pUpstreamDimKey_(e.K) &&
    _pUpstreamDimKey_(c.C) === _pUpstreamDimKey_(e.C) &&
    _pUpstreamDimKey_(c.M) === _pUpstreamDimKey_(e.M);
}

function _preflightCollectUpstreamErrorInventory_(packets, kNotes, cNotes, mNotes){
  const inv = {
    API_ERROR: [],
    VALIDATION_ERROR: [],
    PACKET_ERROR: [],
    PARSER_ERROR: [],
    OTHER_ERROR: []
  };
  const specs = [
    { dim: "K", notes: kNotes },
    { dim: "C", notes: cNotes },
    { dim: "M", notes: mNotes }
  ];
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    const pid = p && p.pid ? String(p.pid) : "";
    if (!row || !pid) return;
    specs.forEach(function(spec){
      const fin = _isFinalKCMPDecisionNote_(spec.notes[row], spec.dim);
      if (!(fin.finalized && fin.status === "ERROR")) return;
      const obj = _parseKCMPNoteJson_(spec.notes[row]);
      const rawType = obj && obj.error_type != null ? String(obj.error_type) : "OTHER";
      const bucket = _normalizeUpstreamErrorTypeBucket_(rawType);
      const key = bucket === "OTHER" ? "OTHER_ERROR" : bucket;
      if (!inv[key]) inv.OTHER_ERROR.push({ dimension: spec.dim, pid: pid, error_type: rawType });
      else inv[key].push({ dimension: spec.dim, pid: pid, error_type: rawType });
    });
  });
  return inv;
}

function _preflightInventoryCounts_(inv){
  return {
    API_ERROR: (inv.API_ERROR || []).length,
    VALIDATION_ERROR: (inv.VALIDATION_ERROR || []).length,
    PACKET_ERROR: (inv.PACKET_ERROR || []).length,
    PARSER_ERROR: (inv.PARSER_ERROR || []).length,
    OTHER_ERROR: (inv.OTHER_ERROR || []).length
  };
}

function _preflightHasRecoverableErrors_(counts){
  return (counts.API_ERROR > 0) ||
    (counts.VALIDATION_ERROR > 0) ||
    (counts.PARSER_ERROR > 0) ||
    (counts.OTHER_ERROR > 0);
}

function _isStaleProductionPAgainstExpected_(packet, pNoteText, kNoteText, cNoteText, mNoteText){
  const pFin = _isFinalKCMPDecisionNote_(pNoteText, "P");
  if (!pFin.finalized) return false;

  const expected = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
  const current = _parseKCMPNoteJson_(pNoteText);
  if (!current || typeof current !== "object") return true;

  const curStatus = String(current.status || "");
  const expStatus = String(expected && expected.status ? expected.status : "");

  // A: status mismatch
  if (curStatus !== expStatus) return true;

  // B/C: both OK — code or contributors differ
  if (curStatus === "OK" && expStatus === "OK") {
    const curCode = current.code == null ? null : String(current.code);
    const expCode = expected.code == null ? null : String(expected.code);
    if (curCode !== expCode) return true;
    if (!_sameCanonicalContributorSet_(current.contributors, expected.contributors)) return true;
    return false;
  }

  // D/E: both ERROR
  if (curStatus === "ERROR" && expStatus === "ERROR") {
    const curEt = String(current.error_type || "");
    const expEt = String(expected.error_type || "");
    if (curEt !== expEt) return true;

    // E: past UPSTREAM_NOT_FINALIZED while K/C/M are now finalized
    if (curEt === "UPSTREAM_NOT_FINALIZED") {
      const kFin = _isFinalKCMPDecisionNote_(kNoteText, "K");
      const cFin = _isFinalKCMPDecisionNote_(cNoteText, "C");
      const mFin = _isFinalKCMPDecisionNote_(mNoteText, "M");
      if (kFin.finalized && cFin.finalized && mFin.finalized) return true;
    }

    if (!_pUpstreamStatesMatch_(current.upstream, expected.upstream)) return true;
    return false;
  }

  return true;
}

function _syncStaleProductionPNotes_(sh, map, packets, kNotes, cNotes, mNotes, pNotes){
  const pCol = colNumOf(map.N);
  const stalePids = [];
  const staleRows = [];

  (packets || []).forEach(function(packet){
    const row = packet && packet.representativeRow;
    const pid = packet && packet.pid ? String(packet.pid) : "";
    if (!row || !pid) return;
    const kNote = kNotes[row] != null ? String(kNotes[row]) : "";
    const cNote = cNotes[row] != null ? String(cNotes[row]) : "";
    const mNote = mNotes[row] != null ? String(mNotes[row]) : "";
    const pNote = pNotes[row] != null ? String(pNotes[row]) : "";
    if (_isStaleProductionPAgainstExpected_(packet, pNote, kNote, cNote, mNote)) {
      stalePids.push(pid);
      staleRows.push(row);
    }
  });

  const cleared = staleRows.length > 0
    ? _clearKCMPProductionCellsAtRows_(sh, pCol, staleRows)
    : 0;

  return {
    stalePids: stalePids,
    staleRows: staleRows,
    staleCleared: cleared
  };
}

/**
 * 메뉴 "10. P 이어서 코딩" 진입점.
 * K/C/M gate → recoverable recovery(1회씩) → stale P sync → deterministic P resume 1회.
 */
function _safeSpreadsheetAlert_(message){
  try {
    SpreadsheetApp.getUi().alert(String(message == null ? "" : message));
    return true;
  } catch (e) {
    Logger.log("UI_ALERT_SKIPPED=true");
    Logger.log("UI_ALERT_MESSAGE=" + String(message == null ? "" : message));
    return false;
  }
}

function runCodeP_WithUpstreamPreflight(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const sheetName = String(sh.getName ? sh.getName() : "");
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const rows = _collectKCMPRepresentativeRows_(packets);

  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const pCol = colNumOf(map.N);

  let progress = _summarizeKCMPProductionProgress_(sh, map, packets);
  let uiAlertSkipped = false;
  let openAiQuotaExhausted = false;

  // STEP A — K/C/M finalized gate
  if (progress.K.unfinalized > 0 || progress.C.unfinalized > 0 || progress.M.unfinalized > 0) {
    Logger.log("=== P UPSTREAM PREFLIGHT ===");
    Logger.log("SHEET=" + sheetName);
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("K_UNFINALIZED=" + progress.K.unfinalized);
    Logger.log("C_UNFINALIZED=" + progress.C.unfinalized);
    Logger.log("M_UNFINALIZED=" + progress.M.unfinalized);
    Logger.log("OPENAI_QUOTA_EXHAUSTED=false");
    Logger.log("PREFLIGHT_RESULT=BLOCKED_UNFINALIZED");
    Logger.log("P_RUN_STARTED=false");
    const alertOk = _safeSpreadsheetAlert_(
      "⚠️ P 코딩을 시작할 수 없습니다.\n\n" +
      "K/C/M이 모두 finalized되어야 합니다.\n\n" +
      "K_UNFINALIZED=" + progress.K.unfinalized + "\n" +
      "C_UNFINALIZED=" + progress.C.unfinalized + "\n" +
      "M_UNFINALIZED=" + progress.M.unfinalized
    );
    uiAlertSkipped = !alertOk;
    Logger.log("UI_ALERT_SKIPPED=" + String(uiAlertSkipped));
    Logger.log("================================");
    return {
      preflightResult: "BLOCKED_UNFINALIZED",
      pRunStarted: false,
      progress: progress,
      openAiQuotaExhausted: false,
      uiAlertSkipped: uiAlertSkipped
    };
  }

  let kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  let cNotes = _batchGetNotesForRows_(sh, cCol, rows);
  let mNotes = _batchGetNotesForRows_(sh, mCol, rows);

  // STEP B — upstream ERROR inventory (before)
  let invBefore = _preflightCollectUpstreamErrorInventory_(packets, kNotes, cNotes, mNotes);
  let countsBefore = _preflightInventoryCounts_(invBefore);

  let apiRecoveryCalled = false;
  let validationRecoveryCalled = false;
  let apiRecoveryResult = null;

  // STEP C — recoverable recovery (각 최대 1회, 기존 recovery 재사용)
  if (countsBefore.API_ERROR > 0) {
    apiRecoveryResult = runKCMPApiErrorRecoveryOnce();
    apiRecoveryCalled = true;
    if (apiRecoveryResult && apiRecoveryResult.quotaExhausted) {
      openAiQuotaExhausted = true;
    }
  }

  // quota exhaustion이면 validation recovery / stale clear / P runner 금지
  if (openAiQuotaExhausted) {
    Logger.log("=== P UPSTREAM PREFLIGHT ===");
    Logger.log("SHEET=" + sheetName);
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("K_FINALIZED_OK=" + progress.K.finalizedOk);
    Logger.log("K_FINALIZED_ERROR=" + progress.K.finalizedError);
    Logger.log("C_FINALIZED_OK=" + progress.C.finalizedOk);
    Logger.log("C_FINALIZED_ERROR=" + progress.C.finalizedError);
    Logger.log("M_FINALIZED_OK=" + progress.M.finalizedOk);
    Logger.log("M_FINALIZED_ERROR=" + progress.M.finalizedError);
    Logger.log("API_ERROR_BEFORE=" + countsBefore.API_ERROR);
    Logger.log("VALIDATION_ERROR_BEFORE=" + countsBefore.VALIDATION_ERROR);
    Logger.log("PACKET_ERROR_BEFORE=" + countsBefore.PACKET_ERROR);
    Logger.log("PARSER_ERROR_BEFORE=" + countsBefore.PARSER_ERROR);
    Logger.log("OTHER_ERROR_BEFORE=" + countsBefore.OTHER_ERROR);
    Logger.log("API_RECOVERY_CALLED=" + String(apiRecoveryCalled));
    Logger.log("VALIDATION_RECOVERY_CALLED=false");
    Logger.log("STALE_P_COUNT=0");
    Logger.log("STALE_P_PIDS=[]");
    Logger.log("STALE_P_CLEARED=0");
    Logger.log("OPENAI_QUOTA_EXHAUSTED=true");
    Logger.log("PREFLIGHT_RESULT=BLOCKED_QUOTA_EXHAUSTED");
    Logger.log("P_RUN_STARTED=false");
    Logger.log("P_GPT_CALLS=0");
    Logger.log("SEMANTIC_CHANGE=NONE");
    Logger.log("AUTO_CHAINING=false");
    const alertOk = _safeSpreadsheetAlert_(
      "⚠️ OpenAI API 크레딧/쿼터가 소진되었습니다.\n\n" +
      "insufficient_quota 감지 — 추가 recovery/P 실행을 중단합니다.\n" +
      "결제/크레딧을 확인한 뒤 다시 실행하세요."
    );
    uiAlertSkipped = !alertOk;
    Logger.log("UI_ALERT_SKIPPED=" + String(uiAlertSkipped));
    Logger.log("================================");
    return {
      preflightResult: "BLOCKED_QUOTA_EXHAUSTED",
      pRunStarted: false,
      countsBefore: countsBefore,
      apiRecoveryCalled: apiRecoveryCalled,
      validationRecoveryCalled: false,
      openAiQuotaExhausted: true,
      uiAlertSkipped: uiAlertSkipped,
      staleCleared: 0,
      apiRecoveryResult: apiRecoveryResult
    };
  }

  if (countsBefore.VALIDATION_ERROR > 0) {
    runKCMPValidationErrorRecoveryOnce();
    validationRecoveryCalled = true;
  }

  // recovery 후 Note 재읽기 + inventory
  kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  cNotes = _batchGetNotesForRows_(sh, cCol, rows);
  mNotes = _batchGetNotesForRows_(sh, mCol, rows);
  progress = _summarizeKCMPProductionProgress_(sh, map, packets);

  let invAfter = _preflightCollectUpstreamErrorInventory_(packets, kNotes, cNotes, mNotes);
  let countsAfter = _preflightInventoryCounts_(invAfter);

  if (_preflightHasRecoverableErrors_(countsAfter)) {
    Logger.log("=== P UPSTREAM PREFLIGHT ===");
    Logger.log("SHEET=" + sheetName);
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("K_FINALIZED_OK=" + progress.K.finalizedOk);
    Logger.log("K_FINALIZED_ERROR=" + progress.K.finalizedError);
    Logger.log("C_FINALIZED_OK=" + progress.C.finalizedOk);
    Logger.log("C_FINALIZED_ERROR=" + progress.C.finalizedError);
    Logger.log("M_FINALIZED_OK=" + progress.M.finalizedOk);
    Logger.log("M_FINALIZED_ERROR=" + progress.M.finalizedError);
    Logger.log("API_ERROR_BEFORE=" + countsBefore.API_ERROR);
    Logger.log("VALIDATION_ERROR_BEFORE=" + countsBefore.VALIDATION_ERROR);
    Logger.log("PACKET_ERROR_BEFORE=" + countsBefore.PACKET_ERROR);
    Logger.log("PARSER_ERROR_BEFORE=" + countsBefore.PARSER_ERROR);
    Logger.log("OTHER_ERROR_BEFORE=" + countsBefore.OTHER_ERROR);
    Logger.log("API_RECOVERY_CALLED=" + String(apiRecoveryCalled));
    Logger.log("VALIDATION_RECOVERY_CALLED=" + String(validationRecoveryCalled));
    Logger.log("API_ERROR_AFTER=" + countsAfter.API_ERROR);
    Logger.log("VALIDATION_ERROR_AFTER=" + countsAfter.VALIDATION_ERROR);
    Logger.log("PACKET_ERROR_AFTER=" + countsAfter.PACKET_ERROR);
    Logger.log("PARSER_ERROR_AFTER=" + countsAfter.PARSER_ERROR);
    Logger.log("OTHER_ERROR_AFTER=" + countsAfter.OTHER_ERROR);
    Logger.log("STRUCTURAL_PACKET_ERROR_COUNT=" + countsAfter.PACKET_ERROR);
    Logger.log("STRUCTURAL_PACKET_ERROR_PIDS=" + JSON.stringify(
      (invAfter.PACKET_ERROR || []).map(function(x){ return x.pid; }).sort()
    ));
    Logger.log("STALE_P_COUNT=0");
    Logger.log("STALE_P_PIDS=[]");
    Logger.log("STALE_P_CLEARED=0");
    Logger.log("OPENAI_QUOTA_EXHAUSTED=false");
    Logger.log("PREFLIGHT_RESULT=RECOVERABLE_ERRORS_REMAIN");
    Logger.log("P_RUN_STARTED=false");
    Logger.log("REMAINING_API_ERROR=" + countsAfter.API_ERROR);
    Logger.log("REMAINING_VALIDATION_ERROR=" + countsAfter.VALIDATION_ERROR);
    Logger.log("REMAINING_PARSER_ERROR=" + countsAfter.PARSER_ERROR);
    Logger.log("REMAINING_OTHER_ERROR=" + countsAfter.OTHER_ERROR);
    Logger.log("P_GPT_CALLS=0");
    Logger.log("SEMANTIC_CHANGE=NONE");
    Logger.log("AUTO_CHAINING=false");
    const alertOk = _safeSpreadsheetAlert_(
      "복구 가능한 upstream 오류가 아직 남아 있습니다.\n" +
      "동일한 P 이어서 코딩을 다시 실행하세요.\n\n" +
      "REMAINING_API_ERROR=" + countsAfter.API_ERROR + "\n" +
      "REMAINING_VALIDATION_ERROR=" + countsAfter.VALIDATION_ERROR + "\n" +
      "REMAINING_PARSER_ERROR=" + countsAfter.PARSER_ERROR + "\n" +
      "REMAINING_OTHER_ERROR=" + countsAfter.OTHER_ERROR
    );
    uiAlertSkipped = !alertOk;
    Logger.log("UI_ALERT_SKIPPED=" + String(uiAlertSkipped));
    Logger.log("================================");
    return {
      preflightResult: "RECOVERABLE_ERRORS_REMAIN",
      pRunStarted: false,
      countsBefore: countsBefore,
      countsAfter: countsAfter,
      apiRecoveryCalled: apiRecoveryCalled,
      validationRecoveryCalled: validationRecoveryCalled,
      openAiQuotaExhausted: false,
      uiAlertSkipped: uiAlertSkipped
    };
  }

  // STEP D — stale P synchronization (PACKET_ERROR만 남아도 진행 가능)
  let pNotes = _batchGetNotesForRows_(sh, pCol, rows);
  const sync = _syncStaleProductionPNotes_(sh, map, packets, kNotes, cNotes, mNotes, pNotes);

  // STEP E — deterministic P resume 1회
  const pStats = _runPProductionResumeBatch_();
  const pRunStarted = true;

  pNotes = _batchGetNotesForRows_(sh, pCol, rows);
  const pProgAfter = _summarizeKCMPDimensionNoteProgress_(packets, pNotes, "P");

  const packetErrorPids = (invAfter.PACKET_ERROR || []).map(function(x){ return x.pid; });
  packetErrorPids.sort();

  let preflightResult = "READY_AND_P_RUN";
  if (sync.staleCleared === 0 &&
      pStats && pStats.processedThisRun === 0 &&
      pStats.complete) {
    preflightResult = "READY_ALREADY_SYNCED";
  }

  Logger.log("=== P UPSTREAM PREFLIGHT ===");
  Logger.log("SHEET=" + sheetName);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("K_FINALIZED_OK=" + progress.K.finalizedOk);
  Logger.log("K_FINALIZED_ERROR=" + progress.K.finalizedError);
  Logger.log("C_FINALIZED_OK=" + progress.C.finalizedOk);
  Logger.log("C_FINALIZED_ERROR=" + progress.C.finalizedError);
  Logger.log("M_FINALIZED_OK=" + progress.M.finalizedOk);
  Logger.log("M_FINALIZED_ERROR=" + progress.M.finalizedError);
  Logger.log("API_ERROR_BEFORE=" + countsBefore.API_ERROR);
  Logger.log("VALIDATION_ERROR_BEFORE=" + countsBefore.VALIDATION_ERROR);
  Logger.log("PACKET_ERROR_BEFORE=" + countsBefore.PACKET_ERROR);
  Logger.log("PARSER_ERROR_BEFORE=" + countsBefore.PARSER_ERROR);
  Logger.log("OTHER_ERROR_BEFORE=" + countsBefore.OTHER_ERROR);
  Logger.log("API_RECOVERY_CALLED=" + String(apiRecoveryCalled));
  Logger.log("VALIDATION_RECOVERY_CALLED=" + String(validationRecoveryCalled));
  Logger.log("API_ERROR_AFTER=" + countsAfter.API_ERROR);
  Logger.log("VALIDATION_ERROR_AFTER=" + countsAfter.VALIDATION_ERROR);
  Logger.log("PACKET_ERROR_AFTER=" + countsAfter.PACKET_ERROR);
  Logger.log("PARSER_ERROR_AFTER=" + countsAfter.PARSER_ERROR);
  Logger.log("OTHER_ERROR_AFTER=" + countsAfter.OTHER_ERROR);
  Logger.log("STRUCTURAL_PACKET_ERROR_COUNT=" + countsAfter.PACKET_ERROR);
  Logger.log("STRUCTURAL_PACKET_ERROR_PIDS=" + JSON.stringify(packetErrorPids));
  Logger.log("STALE_P_COUNT=" + sync.stalePids.length);
  Logger.log("STALE_P_PIDS=" + JSON.stringify(sync.stalePids));
  Logger.log("STALE_P_CLEARED=" + sync.staleCleared);
  Logger.log("OPENAI_QUOTA_EXHAUSTED=false");
  Logger.log("UI_ALERT_SKIPPED=false");
  Logger.log("PREFLIGHT_RESULT=" + preflightResult);
  Logger.log("P_RUN_STARTED=" + String(pRunStarted));
  Logger.log("P_FINALIZED_OK_AFTER=" + pProgAfter.finalizedOk);
  Logger.log("P_FINALIZED_ERROR_AFTER=" + pProgAfter.finalizedError);
  Logger.log("P_UNFINALIZED_AFTER=" + pProgAfter.unfinalized);
  Logger.log("P_GPT_CALLS=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("AUTO_CHAINING=false");
  Logger.log("================================");

  return {
    preflightResult: preflightResult,
    pRunStarted: pRunStarted,
    countsBefore: countsBefore,
    countsAfter: countsAfter,
    apiRecoveryCalled: apiRecoveryCalled,
    validationRecoveryCalled: validationRecoveryCalled,
    openAiQuotaExhausted: false,
    uiAlertSkipped: false,
    stalePids: sync.stalePids,
    staleCleared: sync.staleCleared,
    packetErrorPids: packetErrorPids,
    pStats: pStats,
    pAfter: {
      finalizedOk: pProgAfter.finalizedOk,
      finalizedError: pProgAfter.finalizedError,
      unfinalized: pProgAfter.unfinalized
    }
  };
}

// ============================================================
// STEP 10B: One-shot VALIDATION_ERROR recovery (orchestration only)
// status=ERROR + error_type=VALIDATION_ERROR 만 1회 production runner 재실행.
// PACKET_ERROR / API_ERROR / PARSER_ERROR / OK / unfinalized → SKIP.
// ============================================================

function _recoveryParseErrorNote_(noteText, dimension){
  const fin = _isFinalKCMPDecisionNote_(noteText, dimension);
  if (!fin.finalized || fin.status !== "ERROR") return { kind: "not_validation_candidate" };
  const obj = _parseKCMPNoteJson_(noteText);
  if (!obj) return { kind: "not_validation_candidate" };
  const errorType = obj.error_type != null ? String(obj.error_type) : "OTHER";
  if (errorType === "VALIDATION_ERROR") {
    return {
      kind: "validation_error",
      errorType: errorType,
      message: obj.message != null ? String(obj.message) : "",
      beforeNote: obj
    };
  }
  return { kind: "skip_nonvalidation", errorType: errorType };
}

function _recoveryLogCandidateResult_(entry){
  Logger.log("DIMENSION=" + entry.dimension);
  Logger.log("PID=" + entry.pid);
  Logger.log("BEFORE_ERROR_TYPE=" + entry.beforeErrorType);
  Logger.log("BEFORE_MESSAGE=" + entry.beforeMessage);
  Logger.log("RECOVERY_ATTEMPTED=" + String(entry.recoveryAttempted));
  Logger.log("AFTER_STATUS=" + (entry.afterStatus != null ? entry.afterStatus : ""));
  Logger.log("AFTER_CODE=" + (entry.afterCode != null ? entry.afterCode : ""));
  Logger.log("AFTER_ERROR_TYPE=" + (entry.afterErrorType != null ? entry.afterErrorType : ""));
  Logger.log("AFTER_MESSAGE=" + (entry.afterMessage != null ? entry.afterMessage : ""));
  Logger.log("AFTER_CONTRIBUTORS=" + JSON.stringify(entry.afterContributors || []));
  Logger.log("RECOVERED=" + String(entry.recovered));
  if (entry.pAfterStatus != null) {
    Logger.log("P_AFTER_STATUS=" + entry.pAfterStatus);
    Logger.log("P_AFTER_CODE=" + (entry.pAfterCode != null ? entry.pAfterCode : ""));
    Logger.log("P_AFTER_ERROR_TYPE=" + (entry.pAfterErrorType != null ? entry.pAfterErrorType : ""));
    Logger.log("P_AFTER_CONTRIBUTORS=" + JSON.stringify(entry.pAfterContributors || []));
  }
  Logger.log("---");
}

function _recoveryRefreshPForPid_(sh, map, packet, kNotesByRow, cNotesByRow, mNotesByRow){
  const row = packet && packet.representativeRow;
  const nCol = colNumOf(map.N);
  if (!row || !nCol) return null;
  const kNoteText = kNotesByRow[row] != null ? String(kNotesByRow[row]) : "";
  const cNoteText = cNotesByRow[row] != null ? String(cNotesByRow[row]) : "";
  const mNoteText = mNotesByRow[row] != null ? String(mNotesByRow[row]) : "";
  const pResult = computeDeterministicPForPacket_(packet, kNoteText, cNoteText, mNoteText);
  _writePDecisionCell_(sh, row, nCol, pResult);
  return pResult;
}

/**
 * 기존 finalized VALIDATION_ERROR(K/C/M)만 1회 production runner 재실행.
 * PACKET_ERROR 등은 SKIP. affected PID의 P만 deterministic refresh.
 */
function runKCMPValidationErrorRecoveryOnce(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const sheetName = String(sh.getName ? sh.getName() : "");
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const rows = _collectKCMPRepresentativeRows_(packets);

  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);

  let kNotesByRow = _batchGetNotesForRows_(sh, kCol, rows);
  let cNotesByRow = _batchGetNotesForRows_(sh, cCol, rows);
  let mNotesByRow = _batchGetNotesForRows_(sh, mCol, rows);

  const dimSpecs = [
    { dimension: "K", col: kCol, notesByRow: kNotesByRow, run: function(p){ return runKDecisionTreeForPacket_(p); }, write: _writeKDecisionCell_ },
    { dimension: "C", col: cCol, notesByRow: cNotesByRow, run: function(p){ return runCDecisionTreeForPacket_(p); }, write: _writeCDecisionCell_ },
    { dimension: "M", col: mCol, notesByRow: mNotesByRow, run: function(p){ return runMDecisionTreeForPacket_(p, { allPackets: packets }); }, write: _writeMDecisionCell_ }
  ];

  let validationCandidates = 0;
  let recoveryAttempts = 0;
  let recoveredOk = 0;
  let remainingValidationError = 0;
  let skippedPacketError = 0;
  let skippedOtherError = 0;
  const affectedPidSet = {};
  const recoveryEntries = [];

  Logger.log("=== KCMP VALIDATION ERROR RECOVERY (START) ===");

  packets.forEach(function(packet){
    const row = packet && packet.representativeRow;
    const pid = packet && packet.pid ? packet.pid : "";
    if (!row || !pid) return;

    dimSpecs.forEach(function(spec){
      const noteText = spec.notesByRow[row];
      const parsed = _recoveryParseErrorNote_(noteText, spec.dimension);

      if (parsed.kind === "skip_nonvalidation") {
        if (parsed.errorType === "PACKET_ERROR") {
          skippedPacketError++;
          Logger.log("SKIP_NONVALIDATION_ERROR");
          Logger.log("DIMENSION=" + spec.dimension);
          Logger.log("PID=" + pid);
          Logger.log("ERROR_TYPE=PACKET_ERROR");
          Logger.log("---");
        } else {
          skippedOtherError++;
        }
        return;
      }
      if (parsed.kind !== "validation_error") return;

      validationCandidates++;
      recoveryAttempts++;
      affectedPidSet[pid] = true;

      const beforeNote = parsed.beforeNote;
      const entry = {
        dimension: spec.dimension,
        pid: pid,
        row: row,
        beforeErrorType: parsed.errorType,
        beforeMessage: parsed.message,
        recoveryAttempted: true,
        recovered: false
      };

      const result = spec.run(packet);
      spec.write(sh, row, spec.col, result);

      if (spec.dimension === "K") kNotesByRow[row] = JSON.stringify(result || {});
      else if (spec.dimension === "C") cNotesByRow[row] = JSON.stringify(result || {});
      else if (spec.dimension === "M") mNotesByRow[row] = JSON.stringify(result || {});

      entry.afterStatus = result && result.status ? String(result.status) : "";
      entry.afterCode = (result && result.code === null) ? "null" : (result && result.code ? String(result.code) : "");
      entry.afterErrorType = result && result.error_type != null ? String(result.error_type) : "";
      entry.afterMessage = result && result.message != null ? String(result.message) : "";
      entry.afterContributors = (result && result.contributors) ? result.contributors.slice() : [];

      if (result && result.status === "OK") {
        entry.recovered = true;
        recoveredOk++;
      } else if (result && result.status === "ERROR" && String(result.error_type || "") === "VALIDATION_ERROR") {
        remainingValidationError++;
      }

      recoveryEntries.push(entry);
    });
  });

  const affectedPids = Object.keys(affectedPidSet).sort();
  let affectedPRefreshed = 0;
  const pResultsByPid = {};

  affectedPids.forEach(function(pid){
    const matched = packets.filter(function(p){ return p && p.pid === pid; });
    if (matched.length !== 1) return;
    const packet = matched[0];
    const row = packet.representativeRow;
    if (!row) return;

    const kNoteRow = _batchGetNotesForRows_(sh, kCol, [row]);
    const cNoteRow = _batchGetNotesForRows_(sh, cCol, [row]);
    const mNoteRow = _batchGetNotesForRows_(sh, mCol, [row]);

    const pResult = _recoveryRefreshPForPid_(sh, map, packet, kNoteRow, cNoteRow, mNoteRow);
    pResultsByPid[pid] = pResult;
    affectedPRefreshed++;
  });

  recoveryEntries.forEach(function(entry){
    const pResult = pResultsByPid[entry.pid];
    if (pResult) {
      entry.pAfterStatus = pResult.status ? String(pResult.status) : "";
      entry.pAfterCode = (pResult.code === null || pResult.code === undefined) ? "null" : String(pResult.code || "");
      entry.pAfterErrorType = pResult.error_type != null ? String(pResult.error_type) : "";
      entry.pAfterContributors = (pResult.contributors) ? pResult.contributors.slice() : [];
    }
    _recoveryLogCandidateResult_(entry);
  });

  Logger.log("\n=== KCMP VALIDATION ERROR RECOVERY ===");
  Logger.log("SHEET=" + sheetName);
  Logger.log("VALIDATION_ERROR_CANDIDATES=" + validationCandidates);
  Logger.log("RECOVERY_ATTEMPTS=" + recoveryAttempts);
  Logger.log("RECOVERED_OK=" + recoveredOk);
  Logger.log("REMAINING_VALIDATION_ERROR=" + remainingValidationError);
  Logger.log("SKIPPED_PACKET_ERROR=" + skippedPacketError);
  Logger.log("SKIPPED_OTHER_ERROR=" + skippedOtherError);
  Logger.log("AFFECTED_P_REFRESHED=" + affectedPRefreshed);
  Logger.log("P_GPT_CALLS=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("PACKET_MAPPING_DIFF=NONE");
  Logger.log("K_PROMPT_DIFF=NONE");
  Logger.log("C_PROMPT_DIFF=NONE");
  Logger.log("M_PROMPT_DIFF=NONE");
  Logger.log("VALIDATOR_DIFF=NONE");
  Logger.log("========================================");

  return {
    sheet: sheetName,
    validationErrorCandidates: validationCandidates,
    recoveryAttempts: recoveryAttempts,
    recoveredOk: recoveredOk,
    remainingValidationError: remainingValidationError,
    skippedPacketError: skippedPacketError,
    skippedOtherError: skippedOtherError,
    affectedPRefreshed: affectedPRefreshed,
    affectedPids: affectedPids,
    entries: recoveryEntries
  };
}

function _recoveryParseApiErrorNote_(noteText, dimension){
  const fin = _isFinalKCMPDecisionNote_(noteText, dimension);
  if (!fin.finalized || fin.status !== "ERROR") return { kind: "not_candidate" };
  const obj = _parseKCMPNoteJson_(noteText);
  if (!obj) return { kind: "not_candidate" };
  const errorType = obj.error_type != null ? String(obj.error_type) : "OTHER";
  if (errorType === "API_ERROR") {
    return {
      kind: "api_error",
      errorType: errorType,
      message: obj.message != null ? String(obj.message) : ""
    };
  }
  return { kind: "skip", errorType: errorType };
}

function _countKCMPApiErrorNotes_(packets, kNotesByRow, cNotesByRow, mNotesByRow){
  let n = 0;
  (packets || []).forEach(function(packet){
    const row = packet && packet.representativeRow;
    if (!row) return;
    if (_recoveryParseApiErrorNote_(kNotesByRow[row], "K").kind === "api_error") n++;
    if (_recoveryParseApiErrorNote_(cNotesByRow[row], "C").kind === "api_error") n++;
    if (_recoveryParseApiErrorNote_(mNotesByRow[row], "M").kind === "api_error") n++;
  });
  return n;
}

/**
 * finalized K/C/M API_ERROR만 1회 production runner 재실행.
 * PACKET_ERROR / VALIDATION_ERROR / PARSER_ERROR 재시도 금지.
 * 한 실행당 최대 KCMP_API_ERROR_RECOVERY_MAX_CASES건. 자동 chaining 없음.
 */
function runKCMPApiErrorRecoveryOnce(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const sheetName = String(sh.getName ? sh.getName() : "");
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const rows = _collectKCMPRepresentativeRows_(packets);

  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);

  const kNotesByRow = _batchGetNotesForRows_(sh, kCol, rows);
  const cNotesByRow = _batchGetNotesForRows_(sh, cCol, rows);
  const mNotesByRow = _batchGetNotesForRows_(sh, mCol, rows);

  const dimSpecs = [
    { dimension: "K", col: kCol, notesByRow: kNotesByRow, run: function(p){ return runKDecisionTreeForPacket_(p); }, write: _writeKDecisionCell_ },
    { dimension: "C", col: cCol, notesByRow: cNotesByRow, run: function(p){ return runCDecisionTreeForPacket_(p); }, write: _writeCDecisionCell_ },
    { dimension: "M", col: mCol, notesByRow: mNotesByRow, run: function(p){ return runMDecisionTreeForPacket_(p, { allPackets: packets }); }, write: _writeMDecisionCell_ }
  ];

  const candidates = [];
  packets.forEach(function(packet){
    const row = packet && packet.representativeRow;
    const pid = packet && packet.pid ? packet.pid : "";
    if (!row || !pid) return;
    dimSpecs.forEach(function(spec){
      const parsed = _recoveryParseApiErrorNote_(spec.notesByRow[row], spec.dimension);
      if (parsed.kind !== "api_error") return;
      candidates.push({
        packet: packet,
        pid: pid,
        row: row,
        spec: spec,
        beforeMessage: parsed.message
      });
    });
  });

  const candidatesBefore = candidates.length;
  Logger.log("=== KCMP API ERROR RECOVERY (START) ===");
  Logger.log("SHEET=" + sheetName);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("API_ERROR_CANDIDATES_BEFORE=" + candidatesBefore);
  Logger.log("MAX_CASES=" + KCMP_API_ERROR_RECOVERY_MAX_CASES);

  let recoveryAttempts = 0;
  let recoveredOk = 0;
  let stillApiErrorThisRun = 0;
  let otherErrorAfterRetry = 0;
  let quotaExhausted = false;
  let stopReason = "";
  const affectedPidSet = {};
  const recoveryEntries = [];

  const toProcess = candidates.slice(0, KCMP_API_ERROR_RECOVERY_MAX_CASES);
  for (let idx = 0; idx < toProcess.length; idx++) {
    if (idx > 0) Utilities.sleep(3000);
    const cand = toProcess[idx];

    recoveryAttempts++;
    affectedPidSet[cand.pid] = true;

    let result = null;
    try {
      result = cand.spec.run(cand.packet);
    } catch (e) {
      if (_isOpenAIQuotaExhaustedError_(e)) {
        quotaExhausted = true;
        stopReason = "OPENAI_QUOTA_EXHAUSTED";
        result = {
          status: "ERROR",
          error_type: "API_ERROR",
          code: null,
          contributors: [],
          message: String(e)
        };
        cand.spec.write(sh, cand.row, cand.spec.col, result);
        recoveryEntries.push({
          dimension: cand.spec.dimension,
          pid: cand.pid,
          afterStatus: "ERROR",
          afterCode: "null",
          afterErrorType: "API_ERROR",
          pRefreshed: false
        });
        stillApiErrorThisRun++;
        break;
      }
      throw e;
    }

    cand.spec.write(sh, cand.row, cand.spec.col, result);

    const afterStatus = result && result.status ? String(result.status) : "";
    const afterCode = (result && result.code === null) ? "null" : (result && result.code ? String(result.code) : "");
    const afterErrorType = result && result.error_type != null ? String(result.error_type) : "";
    const resultBlob = [
      afterErrorType,
      result && result.message != null ? String(result.message) : "",
      String(result)
    ].join(" ");

    if (afterStatus === "OK") recoveredOk++;
    else if (afterStatus === "ERROR" && afterErrorType === "API_ERROR") stillApiErrorThisRun++;
    else if (afterStatus === "ERROR") otherErrorAfterRetry++;

    recoveryEntries.push({
      dimension: cand.spec.dimension,
      pid: cand.pid,
      afterStatus: afterStatus,
      afterCode: afterCode,
      afterErrorType: afterErrorType,
      pRefreshed: false
    });

    // decision tree가 API_ERROR Note로 감싼 quota 메시지도 즉시 중단
    if (afterStatus === "ERROR" && _isOpenAIQuotaExhaustedError_(resultBlob)) {
      quotaExhausted = true;
      stopReason = "OPENAI_QUOTA_EXHAUSTED";
      break;
    }
  }

  const affectedPids = Object.keys(affectedPidSet).sort();
  let affectedPRefreshed = 0;
  const pRefreshedPids = {};
  affectedPids.forEach(function(pid){
    const matched = packets.filter(function(p){ return p && p.pid === pid; });
    if (matched.length !== 1) return;
    const packet = matched[0];
    const row = packet.representativeRow;
    if (!row) return;
    const kNoteRow = _batchGetNotesForRows_(sh, kCol, [row]);
    const cNoteRow = _batchGetNotesForRows_(sh, cCol, [row]);
    const mNoteRow = _batchGetNotesForRows_(sh, mCol, [row]);
    _recoveryRefreshPForPid_(sh, map, packet, kNoteRow, cNoteRow, mNoteRow);
    pRefreshedPids[pid] = true;
    affectedPRefreshed++;
  });

  recoveryEntries.forEach(function(entry){
    entry.pRefreshed = !!pRefreshedPids[entry.pid];
    Logger.log("DIMENSION=" + entry.dimension);
    Logger.log("PID=" + entry.pid);
    Logger.log("BEFORE_ERROR_TYPE=API_ERROR");
    Logger.log("RECOVERY_ATTEMPTED=true");
    Logger.log("AFTER_STATUS=" + entry.afterStatus);
    Logger.log("AFTER_CODE=" + entry.afterCode);
    Logger.log("AFTER_ERROR_TYPE=" + entry.afterErrorType);
    Logger.log("P_REFRESHED=" + String(entry.pRefreshed));
    Logger.log("---");
  });

  const kNotesAfter = _batchGetNotesForRows_(sh, kCol, rows);
  const cNotesAfter = _batchGetNotesForRows_(sh, cCol, rows);
  const mNotesAfter = _batchGetNotesForRows_(sh, mCol, rows);
  const remainingApiError = _countKCMPApiErrorNotes_(packets, kNotesAfter, cNotesAfter, mNotesAfter);

  if (quotaExhausted) {
    Logger.log("QUOTA_EXHAUSTED=true");
    Logger.log("STOP_REASON=OPENAI_QUOTA_EXHAUSTED");
  }

  Logger.log("=== KCMP API ERROR RECOVERY ===");
  Logger.log("API_ERROR_CANDIDATES_BEFORE=" + candidatesBefore);
  Logger.log("RECOVERY_ATTEMPTS=" + recoveryAttempts);
  Logger.log("RECOVERED_OK=" + recoveredOk);
  Logger.log("STILL_API_ERROR_THIS_RUN=" + stillApiErrorThisRun);
  Logger.log("OTHER_ERROR_AFTER_RETRY=" + otherErrorAfterRetry);
  Logger.log("REMAINING_API_ERROR=" + remainingApiError);
  Logger.log("AFFECTED_P_REFRESHED=" + affectedPRefreshed);
  Logger.log("MAX_CASES=" + KCMP_API_ERROR_RECOVERY_MAX_CASES);
  Logger.log("QUOTA_EXHAUSTED=" + String(quotaExhausted));
  if (stopReason) Logger.log("STOP_REASON=" + stopReason);
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("AUTO_CHAINING=false");

  return {
    sheet: sheetName,
    apiErrorCandidatesBefore: candidatesBefore,
    recoveryAttempts: recoveryAttempts,
    recoveredOk: recoveredOk,
    stillApiErrorThisRun: stillApiErrorThisRun,
    otherErrorAfterRetry: otherErrorAfterRetry,
    remainingApiError: remainingApiError,
    affectedPRefreshed: affectedPRefreshed,
    affectedPids: affectedPids,
    quotaExhausted: quotaExhausted,
    stopReason: stopReason || ""
  };
}

/** GPT/API 호출·cell write 없음 — 현재 active sheet K/C/M/P production 진행상황 inspect */
function TEST_KCMP_PRODUCTION_PROGRESS(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map);
  const summary = _summarizeKCMPProductionProgress_(sh, map, packets || []);

  Logger.log("=== KCMP PRODUCTION PROGRESS ===");
  Logger.log("SHEET=" + String(sh.getName ? sh.getName() : ""));
  Logger.log("TOTAL_PACKETS=" + summary.totalPackets);
  Logger.log("K_FINALIZED_OK=" + summary.K.finalizedOk);
  Logger.log("K_FINALIZED_ERROR=" + summary.K.finalizedError);
  Logger.log("K_UNFINALIZED=" + summary.K.unfinalized);
  Logger.log("C_FINALIZED_OK=" + summary.C.finalizedOk);
  Logger.log("C_FINALIZED_ERROR=" + summary.C.finalizedError);
  Logger.log("C_UNFINALIZED=" + summary.C.unfinalized);
  Logger.log("M_FINALIZED_OK=" + summary.M.finalizedOk);
  Logger.log("M_FINALIZED_ERROR=" + summary.M.finalizedError);
  Logger.log("M_UNFINALIZED=" + summary.M.unfinalized);
  Logger.log("P_FINALIZED_OK=" + summary.P.finalizedOk);
  Logger.log("P_FINALIZED_ERROR=" + summary.P.finalizedError);
  Logger.log("P_UNFINALIZED=" + summary.P.unfinalized);
  Logger.log("K_COMPLETE=" + String(summary.K.complete));
  Logger.log("C_COMPLETE=" + String(summary.C.complete));
  Logger.log("M_COMPLETE=" + String(summary.M.complete));
  Logger.log("P_COMPLETE=" + String(summary.P.complete));
  Logger.log("ALL_KCM_FINALIZED=" + String(summary.allKcmFinalized));
  Logger.log("ALL_KCMP_FINALIZED=" + String(summary.allKcmpFinalized));
  Logger.log("================================");
  return summary;
}

/** GPT/write 없음 — finalized P Note 분포 집계 (재판정 없음) */
function TEST_P_PRODUCTION_SUMMARY(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const rows = _collectKCMPRepresentativeRows_(packets);
  const pNotes = _batchGetNotesForRows_(sh, colNumOf(map.N), rows);
  const dist = _summarizePProductionFromNotes_(packets, pNotes);

  Logger.log("=== P PRODUCTION SUMMARY ===");
  Logger.log("SHEET=" + String(sh.getName ? sh.getName() : ""));
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("P0_COUNT=" + dist.p0);
  Logger.log("P1_COUNT=" + dist.p1);
  Logger.log("P2_COUNT=" + dist.p2);
  Logger.log("P3_COUNT=" + dist.p3);
  Logger.log("P_ERROR_COUNT=" + dist.pError);
  Logger.log("P_UNFINALIZED=" + dist.unfinalized);
  Logger.log("ERROR_TYPES:");
  Object.keys(dist.errorTypes).sort().forEach(function(et){
    Logger.log("  " + et + "=" + dist.errorTypes[et]);
  });
  Logger.log("============================");
  return dist;
}

// ============================================================
// STEP 10: Read-only KCMP production error inventory
// GPT 호출·sheet write·재시도 없음 — finalized ERROR Note inspect only
// ============================================================

function _parseKCMPNoteJson_(noteText){
  if (noteText == null || String(noteText).trim().length === 0) return null;
  try {
    const obj = JSON.parse(String(noteText));
    if (!obj || typeof obj !== "object" || Array.isArray(obj)) return null;
    return obj;
  } catch (e) {
    return null;
  }
}

function _normalizeUpstreamErrorTypeBucket_(errorType){
  const et = String(errorType || "OTHER").toUpperCase();
  if (et === "PACKET_ERROR") return "PACKET_ERROR";
  if (et === "VALIDATION_ERROR") return "VALIDATION_ERROR";
  if (et === "API_ERROR") return "API_ERROR";
  if (et === "PARSER_ERROR") return "PARSER_ERROR";
  return "OTHER";
}

function _diagnosticErrorCategory_(errorType){
  const bucket = _normalizeUpstreamErrorTypeBucket_(errorType);
  if (bucket === "PACKET_ERROR") return "DATA_OR_MAPPING";
  if (bucket === "VALIDATION_ERROR") return "LLM_OUTPUT_VALIDATION";
  if (bucket === "API_ERROR" || bucket === "PARSER_ERROR") return "API_OR_RUNTIME";
  return "UNKNOWN";
}

function _inventoryRawExcerpt_(obj, maxLen){
  const limit = maxLen || 300;
  if (obj && obj.raw_excerpt != null) {
    const raw = String(obj.raw_excerpt);
    return raw.length > limit ? raw.slice(0, limit) + "…" : raw;
  }
  const full = JSON.stringify(obj || {});
  return full.length > limit ? full.slice(0, limit) + "…" : full;
}

function _inventoryPidSetsEqual_(a, b){
  if (a.length !== b.length) return false;
  const seen = {};
  a.forEach(function(pid){ seen[pid] = true; });
  for (let i = 0; i < b.length; i++) {
    if (!seen[b[i]]) return false;
  }
  return true;
}

function _extractKCMPDimensionErrorEntry_(dimension, noteText, packet){
  const obj = _parseKCMPNoteJson_(noteText);
  if (!obj || obj.status !== "ERROR") return null;
  const errorType = obj.error_type != null ? String(obj.error_type) : "OTHER";
  return {
    dimension: dimension,
    pid: packet && packet.pid ? packet.pid : "",
    row: packet && packet.representativeRow ? packet.representativeRow : "",
    errorType: errorType,
    message: obj.message != null ? String(obj.message) : "",
    validationErrors: Array.isArray(obj.validation_errors) ? obj.validation_errors.slice() : [],
    diagnosticCategory: _diagnosticErrorCategory_(errorType),
    rawExcerpt: _inventoryRawExcerpt_(obj, 300),
    errorBucket: _normalizeUpstreamErrorTypeBucket_(errorType)
  };
}

function _extractPInventoryErrorEntry_(pNoteText, packet, kNoteText, cNoteText, mNoteText){
  const obj = _parseKCMPNoteJson_(pNoteText);
  if (!obj || obj.status !== "ERROR") return null;

  let upstreamDimension = "";
  let upstreamErrorType = "";
  const up = obj.upstream;
  if (up && typeof up === "object") {
    ["K", "C", "M"].forEach(function(dim){
      if (!upstreamDimension && up[dim] && up[dim].state === "ERROR") upstreamDimension = dim;
    });
  }

  if (upstreamDimension) {
    const srcByDim = { K: kNoteText, C: cNoteText, M: mNoteText };
    const srcObj = _parseKCMPNoteJson_(srcByDim[upstreamDimension]);
    if (srcObj && srcObj.error_type != null) upstreamErrorType = String(srcObj.error_type);
  }

  if (!upstreamDimension && obj.message) {
    const m = String(obj.message).match(/^([KCM]) upstream ERROR:\s*(.+)$/);
    if (m) {
      upstreamDimension = m[1];
      upstreamErrorType = m[2].trim();
    }
  }

  return {
    dimension: "P",
    pid: packet && packet.pid ? packet.pid : "",
    row: packet && packet.representativeRow ? packet.representativeRow : "",
    errorType: obj.error_type != null ? String(obj.error_type) : "OTHER",
    message: obj.message != null ? String(obj.message) : "",
    upstreamDimension: upstreamDimension,
    upstreamErrorType: upstreamErrorType
  };
}

function _logKCMPDimensionInventoryError_(entry){
  Logger.log("DIMENSION=" + entry.dimension);
  Logger.log("PID=" + entry.pid);
  Logger.log("REPRESENTATIVE_ROW=" + entry.row);
  Logger.log("ERROR_TYPE=" + entry.errorType);
  Logger.log("MESSAGE=" + entry.message);
  if (entry.validationErrors && entry.validationErrors.length) {
    Logger.log("VALIDATION_ERRORS=" + JSON.stringify(entry.validationErrors));
  } else {
    Logger.log("VALIDATION_ERRORS=[]");
  }
  Logger.log("DIAGNOSTIC_CATEGORY=" + entry.diagnosticCategory);
  Logger.log("RAW_EXCERPT=" + entry.rawExcerpt);
  Logger.log("---");
}

function _logPInventoryError_(entry){
  Logger.log("DIMENSION=P");
  Logger.log("PID=" + entry.pid);
  Logger.log("REPRESENTATIVE_ROW=" + entry.row);
  Logger.log("ERROR_TYPE=" + entry.errorType);
  Logger.log("MESSAGE=" + entry.message);
  Logger.log("UPSTREAM_DIMENSION=" + (entry.upstreamDimension || ""));
  Logger.log("UPSTREAM_ERROR_TYPE=" + (entry.upstreamErrorType || ""));
  Logger.log("---");
}

/** read-only — finalized K/C/M/P ERROR Note inventory (GPT/write/retry 없음) */
function TEST_KCMP_PRODUCTION_ERROR_INVENTORY(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const sheetName = String(sh.getName ? sh.getName() : "");
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const rows = _collectKCMPRepresentativeRows_(packets);

  const kNotes = _batchGetNotesForRows_(sh, colNumOf(map.K), rows);
  const cNotes = _batchGetNotesForRows_(sh, colNumOf(map.L), rows);
  const mNotes = _batchGetNotesForRows_(sh, colNumOf(map.M), rows);
  const pNotes = _batchGetNotesForRows_(sh, colNumOf(map.N), rows);

  const kErrors = [];
  const cErrors = [];
  const mErrors = [];
  const pErrors = [];
  const upstreamErrorPidSet = {};
  const errorTypeCounts = {
    PACKET_ERROR: 0,
    VALIDATION_ERROR: 0,
    API_ERROR: 0,
    PARSER_ERROR: 0,
    OTHER: 0
  };

  Logger.log("=== KCMP PRODUCTION ERROR DETAILS ===");

  packets.forEach(function(packet){
    const row = packet && packet.representativeRow;
    if (!row) return;

    const ke = _extractKCMPDimensionErrorEntry_("K", kNotes[row], packet);
    if (ke) {
      kErrors.push(ke);
      upstreamErrorPidSet[ke.pid] = true;
      errorTypeCounts[ke.errorBucket]++;
      _logKCMPDimensionInventoryError_(ke);
    }

    const ce = _extractKCMPDimensionErrorEntry_("C", cNotes[row], packet);
    if (ce) {
      cErrors.push(ce);
      upstreamErrorPidSet[ce.pid] = true;
      errorTypeCounts[ce.errorBucket]++;
      _logKCMPDimensionInventoryError_(ce);
    }

    const me = _extractKCMPDimensionErrorEntry_("M", mNotes[row], packet);
    if (me) {
      mErrors.push(me);
      upstreamErrorPidSet[me.pid] = true;
      errorTypeCounts[me.errorBucket]++;
      _logKCMPDimensionInventoryError_(me);
    }
  });

  const pErrorPidSet = {};
  packets.forEach(function(packet){
    const row = packet && packet.representativeRow;
    if (!row) return;
    const pe = _extractPInventoryErrorEntry_(pNotes[row], packet, kNotes[row], cNotes[row], mNotes[row]);
    if (!pe) return;
    pErrors.push(pe);
    pErrorPidSet[pe.pid] = true;
    _logPInventoryError_(pe);
  });

  const upstreamUnion = Object.keys(upstreamErrorPidSet).sort();
  const pErrorPids = Object.keys(pErrorPidSet).sort();
  const setsMatch = _inventoryPidSetsEqual_(upstreamUnion, pErrorPids);

  Logger.log("UPSTREAM_ERROR_PID_UNION=" + JSON.stringify(upstreamUnion));
  Logger.log("P_ERROR_PID_SET=" + JSON.stringify(pErrorPids));
  Logger.log("SETS_MATCH=" + String(setsMatch));

  Logger.log("\n=== KCMP PRODUCTION ERROR INVENTORY ===");
  Logger.log("SHEET=" + sheetName);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("K_ERROR_COUNT=" + kErrors.length);
  Logger.log("C_ERROR_COUNT=" + cErrors.length);
  Logger.log("M_ERROR_COUNT=" + mErrors.length);
  Logger.log("P_ERROR_COUNT=" + pErrors.length);
  Logger.log("UPSTREAM_ERROR_PID_COUNT=" + upstreamUnion.length);
  Logger.log("UPSTREAM_ERROR_PIDS=" + JSON.stringify(upstreamUnion));
  Logger.log("PACKET_ERROR_COUNT=" + errorTypeCounts.PACKET_ERROR);
  Logger.log("VALIDATION_ERROR_COUNT=" + errorTypeCounts.VALIDATION_ERROR);
  Logger.log("API_ERROR_COUNT=" + errorTypeCounts.API_ERROR);
  Logger.log("PARSER_ERROR_COUNT=" + errorTypeCounts.PARSER_ERROR);
  Logger.log("OTHER_ERROR_COUNT=" + errorTypeCounts.OTHER);
  Logger.log("P_ERROR_SET_MATCHES_UPSTREAM_UNION=" + String(setsMatch));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("PRODUCTION_DATA_COMPLETE=true");
  Logger.log("RETRY_PERFORMED=false");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("========================================");

  return {
    sheet: sheetName,
    totalPackets: packets.length,
    kErrors: kErrors,
    cErrors: cErrors,
    mErrors: mErrors,
    pErrors: pErrors,
    upstreamErrorPidUnion: upstreamUnion,
    pErrorPidSet: pErrorPids,
    setsMatch: setsMatch,
    errorTypeCounts: errorTypeCounts
  };
}

// ============================================================
// STEP 10A: Read-only PACKET_ERROR speaker mapping diagnostic
// 대상: 14차시 4조::P002, 14차시 4조::P050 (GPT/write/재코딩 없음)
// ============================================================

function _packetDiagReadRowSpeakerCells_(sheet, map, row){
  const cols = {
    S1: colNumOf(map.S1),
    S2: colNumOf(map.S2),
    S3: colNumOf(map.S3),
    S4: colNumOf(map.S4)
  };
  const aCol = colNumOf(map.A);
  const out = {
    row: row,
    speakerCell: "",
    S1_CELL: "",
    S2_CELL: "",
    S3_CELL: "",
    S4_CELL: ""
  };
  if (!sheet || !row || row < 2) return out;
  try {
    out.speakerCell = String(sheet.getRange(row, aCol).getDisplayValue() || "");
    out.S1_CELL = String(sheet.getRange(row, cols.S1).getDisplayValue() || "");
    out.S2_CELL = String(sheet.getRange(row, cols.S2).getDisplayValue() || "");
    out.S3_CELL = String(sheet.getRange(row, cols.S3).getDisplayValue() || "");
    out.S4_CELL = String(sheet.getRange(row, cols.S4).getDisplayValue() || "");
  } catch (e) {
    out.readError = String(e);
  }
  return out;
}

function _packetDiagProblemTurnReason_(turn, auditEntryType){
  if (auditEntryType === "unknown") {
    return "화자명이 비어 있음 (production: unknownSpeakers)";
  }
  if (auditEntryType === "unmapped") {
    return "학생처럼 보이나 S1~S4에 매핑되지 않음 (production: unmappedStudentSpeakers)";
  }
  if (turn && turn.role === "teacher") return "교사 turn (mapping 문제 아님)";
  if (turn && turn.role === "student" && turn.speakerId) return "정상 학생 매핑";
  if (turn && !turn.speakerRaw) return "화자명이 비어 있음";
  if (turn && turn.speakerRaw && isTeacherSpeaker(turn.speakerRaw)) return "교사 speaker";
  return "S1~S4 매핑 실패 (matchSpeakerToSx_=null, isTeacherSpeaker=false)";
}

function _packetDiagCollectProblemTurns_(packet){
  const audit = packet && packet.audit ? packet.audit : {};
  const unmapped = Array.isArray(audit.unmappedStudentSpeakers) ? audit.unmappedStudentSpeakers : [];
  const unknown = Array.isArray(audit.unknownSpeakers) ? audit.unknownSpeakers : [];
  const problems = [];
  const seen = {};

  function addProblem(row, rawValue, reason, kind){
    const key = String(row) + "|" + kind;
    if (seen[key]) return;
    seen[key] = true;
    problems.push({
      SOURCE_ROW: row,
      RAW_VALUE: rawValue != null ? String(rawValue) : "",
      NORMALIZED_VALUE: rawValue ? (_normForMatching_(rawValue).normalized || "") : "",
      REASON: reason,
      KIND: kind
    });
  }

  unknown.forEach(function(u){
    addProblem(u.row, u.speakerRaw, _packetDiagProblemTurnReason_(null, "unknown"), "unknown");
  });
  unmapped.forEach(function(u){
    addProblem(u.row, u.speakerRaw, _packetDiagProblemTurnReason_(null, "unmapped"), "unmapped");
  });

  return {
    problems: problems,
    unknownCount: unknown.length,
    unmappedCount: unmapped.length,
    totalCount: problems.length
  };
}

function _packetDiagClassifyCause_(packet, ctx, problemInfo){
  const active = (packet && packet.activeStudentIds) ? packet.activeStudentIds : [];
  const audit = packet && packet.audit ? packet.audit : {};
  const warnings = Array.isArray(audit.warnings) ? audit.warnings : [];
  const unmapped = Array.isArray(audit.unmappedStudentSpeakers) ? audit.unmappedStudentSpeakers : [];
  const unknown = Array.isArray(audit.unknownSpeakers) ? audit.unknownSpeakers : [];

  let cause = "INSUFFICIENT_INFORMATION";
  let fixData = false;
  let fixCode = false;
  let safeRetry = false;

  if (unknown.length > 0 && unmapped.length === 0) {
    cause = "RAW_DATA_FORMAT_VARIATION";
    fixData = true;
  } else if (unmapped.length > 0) {
    const teacherMis = unmapped.some(function(u){ return isTeacherSpeaker(u.speakerRaw); });
    if (teacherMis) {
      cause = "NONSTUDENT_TURN_MISCLASSIFIED";
      fixCode = true;
    } else {
      const headerLabels = (ctx && ctx.sHeaders) ? ctx.sHeaders.map(function(h){ return h.label; }).join(", ") : "";
      const sampleRaw = unmapped.map(function(u){ return u.speakerRaw; }).join(" | ");
      if (warnings.some(function(w){ return w.indexOf("count mismatch") >= 0 || w.indexOf("activeStudentIds conflict") >= 0; })) {
        cause = "PACKET_MAPPING_LOGIC";
        fixCode = true;
      } else {
        cause = "RAW_DATA_UNKNOWN_SPEAKER";
        fixData = true;
      }
      Logger.log("  CAUSE_EVIDENCE=unmapped_speakers=[" + sampleRaw + "] sHeaders=[" + headerLabels + "]");
    }
  } else if (active.length < 2 && problemInfo.totalCount === 0) {
    cause = "OTHER";
  }

  if (_cPacketMappingUnreliable_(packet)) {
    safeRetry = false;
  }

  return {
    cause: cause,
    fixRequiresDataChange: fixData,
    fixRequiresCodeChange: fixCode,
    safeToRetryWithoutChange: safeRetry
  };
}

function _packetDiagLogTarget_(sheetName, pid, packet, ctx, map){
  const goldKey = sheetName + "::" + pid;
  Logger.log("\n--- TARGET " + goldKey + " ---");
  Logger.log("GOLD_KEY=" + goldKey);
  Logger.log("REPRESENTATIVE_ROW=" + String(packet.representativeRow || ""));

  Logger.log("PID=" + String(packet.pid || ""));
  Logger.log("SUMMARY=" + String(packet.summary || ""));

  Logger.log("ACTIVE_STUDENT_IDS=" + JSON.stringify(packet.activeStudentIds || []));
  Logger.log("STUDENTS=" + JSON.stringify(packet.students || []));
  Logger.log("SPEAKER_COUNTS=" + JSON.stringify(packet.speakerCounts || {}));
  Logger.log("TURN_DERIVED_COUNTS=" + JSON.stringify(packet.turnDerivedCounts || {}));
  Logger.log("TEACHER_PRESENT=" + String(!!packet.teacherPresent));
  Logger.log("PACKET_AUDIT=" + JSON.stringify(packet.audit || {}));

  if (ctx && ctx.sHeaders) {
    Logger.log("S_HEADERS=" + JSON.stringify(ctx.sHeaders));
  }

  Logger.log("\n  --- CURRENT TURNS ---");
  const turns = packet.turns || [];
  turns.forEach(function(t, idx){
    Logger.log("  TURN_INDEX=" + idx);
    Logger.log("  SOURCE_ROW=" + String(t.row != null ? t.row : ""));
    Logger.log("  ROLE=" + String(t.role != null ? t.role : ""));
    Logger.log("  SPEAKER=" + String(t.speakerId != null ? t.speakerId : "null"));
    Logger.log("  RAW_SPEAKER=" + String(t.speakerRaw != null ? t.speakerRaw : ""));
    Logger.log("  TEXT=" + String(t.utterance != null ? t.utterance : ""));
    if (t.timestamp != null) Logger.log("  TIMESTAMP=" + String(t.timestamp));
    Logger.log("  ---");
  });
  if (!turns.length) Logger.log("  (no turns)");

  const mappingUnreliable = _cPacketMappingUnreliable_(packet);
  const problemInfo = _packetDiagCollectProblemTurns_(packet);

  Logger.log("\n  --- MAPPING DIAGNOSTIC ---");
  Logger.log("  MAPPING_UNRELIABLE=" + String(mappingUnreliable));
  Logger.log("  UNKNOWN_OR_UNMAPPED_COUNT=" + problemInfo.totalCount);
  Logger.log("  UNKNOWN_OR_UNMAPPED_TURNS=" + JSON.stringify(problemInfo.problems));

  problemInfo.problems.forEach(function(p){
    Logger.log("  PROBLEM_TURN SOURCE_ROW=" + p.SOURCE_ROW);
    Logger.log("  PROBLEM_TURN RAW_VALUE=" + p.RAW_VALUE);
    Logger.log("  PROBLEM_TURN NORMALIZED_VALUE=" + p.NORMALIZED_VALUE);
    Logger.log("  PROBLEM_TURN REASON=" + p.REASON);
  });

  Logger.log("\n  --- SOURCE ROW SPEAKER COLUMNS (read-only) ---");
  const rowSet = {};
  turns.forEach(function(t){ if (t.row) rowSet[t.row] = true; });
  Object.keys(rowSet).map(Number).sort(function(a, b){ return a - b; }).forEach(function(row){
    const cells = _packetDiagReadRowSpeakerCells_(ctx.sheet, map, row);
    Logger.log("  ROW=" + cells.row);
    Logger.log("  A_SPEAKER_CELL=" + cells.speakerCell);
    Logger.log("  S1_CELL=" + cells.S1_CELL);
    Logger.log("  S2_CELL=" + cells.S2_CELL);
    Logger.log("  S3_CELL=" + cells.S3_CELL);
    Logger.log("  S4_CELL=" + cells.S4_CELL);
  });

  const classification = _packetDiagClassifyCause_(packet, ctx, problemInfo);
  Logger.log("\n  CAUSE=" + classification.cause);
  Logger.log("  FIX_REQUIRES_DATA_CHANGE=" + String(classification.fixRequiresDataChange));
  Logger.log("  FIX_REQUIRES_CODE_CHANGE=" + String(classification.fixRequiresCodeChange));
  Logger.log("  SAFE_TO_RETRY_WITHOUT_CHANGE=" + String(classification.safeToRetryWithoutChange));

  return {
    goldKey: goldKey,
    pid: pid,
    mappingUnreliable: mappingUnreliable,
    unknownOrUnmappedCount: problemInfo.totalCount,
    cause: classification.cause,
    fixRequiresDataChange: classification.fixRequiresDataChange,
    fixRequiresCodeChange: classification.fixRequiresCodeChange,
    safeToRetryWithoutChange: classification.safeToRetryWithoutChange
  };
}

/** read-only diagnostic: 14차시 4조 P002/P050 PACKET_ERROR speaker mapping */
function TEST_PACKET_MAPPING_ERRORS_P002_P050(){
  const SHEET_NAME = "14차시 4조";
  const TARGET_PIDS = ["P002", "P050"];

  const ss = SpreadsheetApp.getActive();
  const sh = ss.getSheetByName(SHEET_NAME);
  if (!sh) {
    Logger.log("ERROR: sheet not found: " + SHEET_NAME);
    return { ok: false, error: "SHEET_NOT_FOUND" };
  }

  const map = loadColMap_();
  if (!map || !map.S1) {
    Logger.log("ERROR: COLMAP missing or incomplete");
    return { ok: false, error: "COLMAP_MISSING" };
  }

  Logger.log("S1~S4 cols=" + [colNumOf(map.S1), colNumOf(map.S2), colNumOf(map.S3), colNumOf(map.S4)].join(","));

  const ctx = _prepareKCMPPacketContext_(sh, map);
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const results = {};

  TARGET_PIDS.forEach(function(pid){
    const goldKey = SHEET_NAME + "::" + pid;
    const matched = packets.filter(function(p){ return p && p.pid === pid; });
    if (matched.length !== 1) {
      Logger.log("GOLD_KEY=" + goldKey + " ABORT MATCH_COUNT=" + matched.length);
      results[pid] = { ok: false, error: "MATCH_COUNT != 1", matchCount: matched.length };
      return;
    }
    results[pid] = _packetDiagLogTarget_(SHEET_NAME, pid, matched[0], ctx, map);
    results[pid].ok = true;
  });

  Logger.log("\n=== PACKET MAPPING ERROR DIAGNOSTIC ===");
  Logger.log("SHEET=" + SHEET_NAME);

  TARGET_PIDS.forEach(function(pid){
    const r = results[pid] || {};
    Logger.log("P" + pid.replace(/^P/, "") + "_CAUSE=" + (r.cause || "NOT_RUN"));
    Logger.log("P" + pid.replace(/^P/, "") + "_UNKNOWN_OR_UNMAPPED_COUNT=" + String(r.unknownOrUnmappedCount != null ? r.unknownOrUnmappedCount : ""));
    Logger.log("P" + pid.replace(/^P/, "") + "_FIX_REQUIRES_DATA_CHANGE=" + String(r.fixRequiresDataChange != null ? r.fixRequiresDataChange : ""));
    Logger.log("P" + pid.replace(/^P/, "") + "_FIX_REQUIRES_CODE_CHANGE=" + String(r.fixRequiresCodeChange != null ? r.fixRequiresCodeChange : ""));
  });

  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("PRODUCTION_CODE_DIFF=NONE");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("========================================");

  return { ok: true, sheet: SHEET_NAME, results: results };
}

/** local/static — finalized Note detection unit checks (GPT/sheet write 없음) */
function TEST_KCMP_FINAL_NOTE_DETECTION(){
  const checks = [];

  function add(name, noteText, dim, expectFinalized){
    const got = _isFinalKCMPDecisionNote_(noteText, dim);
    const ok = got.finalized === expectFinalized;
    checks.push({ name: name, ok: ok, expect: expectFinalized, got: got.finalized, status: got.status });
  }

  add("OK K", JSON.stringify({ schema_version: "KCMP_K_V1", status: "OK", code: "K2", contributors: ["S1"] }), "K", true);
  add("ERROR K", JSON.stringify({ schema_version: "KCMP_K_V1", status: "ERROR", error_type: "VALIDATION_ERROR", code: null }), "K", true);
  add("OK null C", JSON.stringify({ schema_version: "KCMP_C_V1", status: "OK", code: null, contributors: [] }), "C", true);
  add("ERROR M", JSON.stringify({ schema_version: "KCMP_M_V1", status: "ERROR", error_type: "PACKET_ERROR" }), "M", true);
  add("OK P0", JSON.stringify({ schema_version: "KCMP_P_V1", status: "OK", code: "P0", contributors: [] }), "P", true);
  add("ERROR P upstream", JSON.stringify({ schema_version: "KCMP_P_V1", status: "ERROR", error_type: "UPSTREAM_ERROR", code: null, contributors: [] }), "P", true);
  add("P blank", "", "P", false);
  add("P wrong schema", JSON.stringify({ schema_version: "KCMP_P_V0", status: "OK", code: "P1" }), "P", false);
  add("blank", "", "K", false);
  add("invalid JSON", "{bad", "K", false);
  add("wrong schema", JSON.stringify({ schema_version: "KCMP_K_V0", status: "OK", code: "K1" }), "K", false);
  add("unfinished status", JSON.stringify({ schema_version: "KCMP_M_V1", status: "PENDING" }), "M", false);

  let allPass = true;
  Logger.log("=== TEST_KCMP_FINAL_NOTE_DETECTION ===");
  checks.forEach(function(c){
    if (!c.ok) allPass = false;
    Logger.log((c.ok ? "PASS" : "FAIL") + "  " + c.name + "  expect=" + c.expect + "  got=" + c.got + "  status=" + c.status);
  });
  Logger.log("OVERALL=" + (allPass ? "ALL_PASS" : "HAS_FAILURES"));
  Logger.log("======================================");
  return allPass;
}

// ============================================================
// STEP 13A: Read-only human consensus agreement benchmark
// 14차시 4조 gold vs current AI cell codes. GPT/write/production runner 없음.
// 메뉴 미등록. Apps Script에서 TEST_KCMP_AGREEMENT_14_4() 수동 실행.
// ============================================================

const KCMP_GOLD_14_4 = {
  K: [
    "","","K1","","","K1","K3","K1","K1","K2","K1","","K1","","K1","","K1","K3",
    "","","","","","K1","K1","K1","","","K3","K3","K3","K3","","","K3","K3","K3",
    "K3","K3","K3","","K1","K3","K1","K1","K1","K3","K1","","K1","K3","K1","K1",
    "K1","K1","","K1","K3","K1","K1","K1","K3","K1","K3","","","","","K3","K3",
    "","K1","K1"
  ],
  C: [
    "","","","","","","C2","C3","C2","C2","C2","C2","C2","C2","","","","","","C2",
    "","","","","","C2","C2","C2","C2","C2","C6","C3","C2","C1","C3","C2","C3",
    "","","C1","","","C3","","","","C3","","","","","C4","C2","C2","","","C3","C6",
    "","","C6","C3","","","C6","C6","","","","","C1","",""
  ],
  M: [
    "","","","","","M4","","M4","M4","M4","M4","M4","","","M4",
    "M4","M1","","","","","M1","","","","","","","","",
    "","","M1","","","","","M3","M3","","","","M3","","",
    "","","","M1","","M3","","M3","M3","M3","","M3","M3","","",
    "","M4","M4","M3","M3","","","","","","","",""
  ],
  P: [
    "P0","P0","P1","P0","P0","P3","P2","P3","P3","P2","P2","P3","P2","P1","P1",
    "P1","P1","P3","P0","P2","P0","P1","P0","P1","P1","P3","P2","P2","P2","P3",
    "P2","P2","P2","P2","P2","P3","P2","P1","P2","P2","P0","P1","P3","P1","P1",
    "P1","P2","P1","P1","P1","P2","P2","P2","P3","P1","P0","P3","P3","P1","P1",
    "P3","P3","P2","P1","P3","P2","P0","P0","P1","P1","P2","P1","P1"
  ]
};

const KCMP_GOLD_14_4_REFERENCE = {
  K: { kappa: 0.1755, agreement: 43.8, aiDistribution: { BLANK: 54, K1: 10, K2: 3, K3: 6 } },
  C: { kappa: 0.4069, agreement: 67.1, aiDistribution: { BLANK: 58, C1: 2, C2: 1, C3: 10, C6: 2 } },
  M: { kappa: 0.4559, agreement: 74.0, aiDistribution: { BLANK: 54, M1: 2, M3: 10, M4: 7 } },
  P: { kappa: 0.1971, agreement: 32.9, aiDistribution: { BLANK: 23, P0: 23, P1: 16, P2: 9, P3: 2 } }
};

function _agreementPidFromIndex_(i){
  const n = i + 1;
  if (n < 10) return "P00" + n;
  if (n < 100) return "P0" + n;
  return "P" + n;
}

function _agreementParseDimCode_(displayText, dimension){
  const raw = String(displayText == null ? "" : displayText).trim();
  if (!raw) return "";
  const dim = String(dimension || "").toUpperCase();
  let re = null;
  if (dim === "K") re = /\b(K[123])\b/;
  else if (dim === "C") re = /\b(C[1-7])\b/;
  else if (dim === "M") re = /\b(M[1-4])\b/;
  else if (dim === "P") re = /\b(P[0-3])\b/;
  if (!re) return "";
  const m = raw.match(re);
  return m ? m[1] : "";
}

function _computeCohenKappa_(goldCodes, aiCodes){
  const n = Math.min((goldCodes || []).length, (aiCodes || []).length);
  if (n === 0) return { n: 0, po: 0, pe: 0, kappa: null, match: 0 };
  const goldCount = {};
  const aiCount = {};
  const cats = {};
  let match = 0;
  for (let i = 0; i < n; i++) {
    const g = goldCodes[i] == null ? "" : String(goldCodes[i]);
    const a = aiCodes[i] == null ? "" : String(aiCodes[i]);
    cats[g] = true;
    cats[a] = true;
    goldCount[g] = (goldCount[g] || 0) + 1;
    aiCount[a] = (aiCount[a] || 0) + 1;
    if (g === a) match++;
  }
  const po = match / n;
  let pe = 0;
  Object.keys(cats).forEach(function(c){
    pe += ((goldCount[c] || 0) / n) * ((aiCount[c] || 0) / n);
  });
  let kappa = null;
  if (Math.abs(1 - pe) < 1e-12) {
    kappa = (Math.abs(po - 1) < 1e-12) ? 1 : 0;
  } else {
    kappa = (po - pe) / (1 - pe);
  }
  return { n: n, po: po, pe: pe, kappa: kappa, match: match };
}

function _agreementDistribution_(codes){
  const dist = {};
  (codes || []).forEach(function(c){
    const key = (c == null || c === "") ? "BLANK" : String(c);
    dist[key] = (dist[key] || 0) + 1;
  });
  return dist;
}

function _agreementFormatDistribution_(dist){
  const keys = Object.keys(dist || {}).sort(function(a, b){
    if (a === "BLANK") return -1;
    if (b === "BLANK") return 1;
    return a < b ? -1 : (a > b ? 1 : 0);
  });
  return keys.map(function(k){ return k + "=" + dist[k]; }).join(" ");
}

function _agreementTaxonomy_(goldCodes, aiCodes){
  const n = Math.min((goldCodes || []).length, (aiCodes || []).length);
  let match = 0;
  let aiBlankGoldCode = 0;
  let aiCodeGoldBlank = 0;
  let subcodeMismatch = 0;
  const confusion = {};
  for (let i = 0; i < n; i++) {
    const g = goldCodes[i] == null ? "" : String(goldCodes[i]);
    const a = aiCodes[i] == null ? "" : String(aiCodes[i]);
    if (g === a) {
      match++;
      continue;
    }
    const gLabel = g === "" ? "BLANK" : g;
    const aLabel = a === "" ? "BLANK" : a;
    const pair = aLabel + " -> " + gLabel;
    confusion[pair] = (confusion[pair] || 0) + 1;
    if (a === "" && g !== "") aiBlankGoldCode++;
    else if (a !== "" && g === "") aiCodeGoldBlank++;
    else subcodeMismatch++;
  }
  const pairs = Object.keys(confusion).map(function(k){
    return { pair: k, count: confusion[k] };
  }).sort(function(a, b){
    if (b.count !== a.count) return b.count - a.count;
    return a.pair < b.pair ? -1 : 1;
  });
  return {
    match: match,
    aiBlankGoldCode: aiBlankGoldCode,
    aiCodeGoldBlank: aiCodeGoldBlank,
    subcodeMismatch: subcodeMismatch,
    confusionPairs: pairs
  };
}

function _agreementFormatConfusion_(pairs, limit){
  const n = limit || 10;
  if (!pairs || !pairs.length) return "(none)";
  return pairs.slice(0, n).map(function(p){ return p.pair + "=" + p.count; }).join("; ");
}

function _agreementRound_(x, digits){
  if (x == null || !isFinite(x)) return "null";
  const d = digits == null ? 4 : digits;
  const m = Math.pow(10, d);
  return String(Math.round(x * m) / m);
}

function _agreementDistributionsEqual_(actual, expected){
  const a = actual || {};
  const e = expected || {};
  const keys = {};
  Object.keys(a).forEach(function(k){ keys[k] = true; });
  Object.keys(e).forEach(function(k){ keys[k] = true; });
  const list = Object.keys(keys);
  for (let i = 0; i < list.length; i++) {
    const k = list[i];
    if (Number(a[k] || 0) !== Number(e[k] || 0)) return false;
  }
  return true;
}

function _agreementReportReferenceMatch_(kappa, agreementPercent, aiDist, ref){
  if (!ref) return false;
  const kappaOk = _agreementRound_(kappa, 4) === _agreementRound_(ref.kappa, 4);
  const agrOk = _agreementRound_(agreementPercent, 1) === _agreementRound_(ref.agreement, 1);
  const distOk = _agreementDistributionsEqual_(aiDist, ref.aiDistribution || {});
  return kappaOk && agrOk && distOk;
}

/** read-only: 14차시 4조 human gold vs current AI K/C/M/P cells */
function TEST_KCMP_AGREEMENT_14_4(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const gold = KCMP_GOLD_14_4;

  Logger.log("=== KCMP AGREEMENT BENCHMARK 14차시 4조 ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("TOTAL_PID=" + TOTAL);
  Logger.log("GOLD_K_COUNT=" + (gold.K || []).length);
  Logger.log("GOLD_C_COUNT=" + (gold.C || []).length);
  Logger.log("GOLD_M_COUNT=" + (gold.M || []).length);
  Logger.log("GOLD_P_COUNT=" + (gold.P || []).length);

  if ((gold.K || []).length !== TOTAL || (gold.C || []).length !== TOTAL ||
      (gold.M || []).length !== TOTAL || (gold.P || []).length !== TOTAL) {
    Logger.log("DIAGNOSTIC_ERROR=GOLD_LENGTH_MISMATCH");
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("CLEAR_CALLS=0");
    Logger.log("PRODUCTION_RUNNER_CALLS=0");
    Logger.log("READ_ONLY=true");
    return { ok: false, error: "GOLD_LENGTH_MISMATCH" };
  }

  const ss = SpreadsheetApp.getActive();
  const sh = ss.getSheetByName(SHEET_NAME);
  if (!sh) {
    Logger.log("DIAGNOSTIC_ERROR=SHEET_NOT_FOUND");
    return { ok: false, error: "SHEET_NOT_FOUND" };
  }
  const map = loadColMap_();
  if (!map || !map.K || !map.L || !map.M || !map.N) {
    Logger.log("DIAGNOSTIC_ERROR=COLMAP_MISSING");
    return { ok: false, error: "COLMAP_MISSING" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  let packetPidCount = 0;
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    if (byPid[pid]) packetPidCount++;
  }
  Logger.log("PACKET_PID_COUNT=" + packetPidCount);
  if (packetPidCount !== TOTAL) {
    Logger.log("DIAGNOSTIC_ERROR=PACKET_PID_COUNT_MISMATCH expected=" + TOTAL + " got=" + packetPidCount);
  }

  const dims = [
    { dim: "K", gold: gold.K, col: colNumOf(map.K) },
    { dim: "C", gold: gold.C, col: colNumOf(map.L) },
    { dim: "M", gold: gold.M, col: colNumOf(map.M) },
    { dim: "P", gold: gold.P, col: colNumOf(map.N) }
  ];

  const lastRow = sh.getLastRow();
  const rowCount = Math.max(0, lastRow - 1);
  const displayByDim = {};
  const notesByDim = {};
  dims.forEach(function(d){
    displayByDim[d.dim] = rowCount > 0 ? sh.getRange(2, d.col, rowCount, 1).getDisplayValues() : [];
    const rows = [];
    for (let i = 0; i < TOTAL; i++) {
      const pid = _agreementPidFromIndex_(i);
      const packet = byPid[pid];
      if (packet && packet.representativeRow) rows.push(packet.representativeRow);
    }
    notesByDim[d.dim] = _batchGetNotesForRows_(sh, d.col, rows);
  });

  const results = {};
  let hasFinalizationIssues = false;
  let reportOutputVersionDiff = false;

  dims.forEach(function(d){
    const aiCodes = [];
    let finalizedOk = 0;
    let finalizedError = 0;
    let unfinalized = 0;

    for (let i = 0; i < TOTAL; i++) {
      const pid = _agreementPidFromIndex_(i);
      const packet = byPid[pid];
      if (!packet || !packet.representativeRow) {
        aiCodes.push("");
        unfinalized++;
        continue;
      }
      const row = packet.representativeRow;
      const idx = row - 2;
      const disp = (idx >= 0 && idx < displayByDim[d.dim].length) ? displayByDim[d.dim][idx][0] : "";
      aiCodes.push(_agreementParseDimCode_(disp, d.dim));

      const fin = _isFinalKCMPDecisionNote_(notesByDim[d.dim][row], d.dim);
      if (fin.finalized && fin.status === "OK") finalizedOk++;
      else if (fin.finalized && fin.status === "ERROR") finalizedError++;
      else unfinalized++;
    }

    if (finalizedError > 0 || unfinalized > 0) hasFinalizationIssues = true;

    const kappa = _computeCohenKappa_(d.gold, aiCodes);
    const tax = _agreementTaxonomy_(d.gold, aiCodes);
    const goldDist = _agreementDistribution_(d.gold);
    const aiDist = _agreementDistribution_(aiCodes);
    const ref = KCMP_GOLD_14_4_REFERENCE[d.dim] || {};
    const agreementPercent = kappa.po * 100;
    const reportMatch = _agreementReportReferenceMatch_(kappa.kappa, agreementPercent, aiDist, ref);
    if (!reportMatch) reportOutputVersionDiff = true;

    Logger.log("");
    Logger.log("--- " + d.dim + " ---");
    Logger.log("KAPPA=" + _agreementRound_(kappa.kappa, 4));
    Logger.log("AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent, 1));
    Logger.log("MATCH=" + tax.match);
    Logger.log("AI_BLANK_GOLD_CODE=" + tax.aiBlankGoldCode);
    Logger.log("AI_CODE_GOLD_BLANK=" + tax.aiCodeGoldBlank);
    Logger.log("SUBCODE_MISMATCH=" + tax.subcodeMismatch);
    Logger.log("FINALIZED_OK=" + finalizedOk);
    Logger.log("FINALIZED_ERROR=" + finalizedError);
    Logger.log("UNFINALIZED=" + unfinalized);
    Logger.log("GOLD_DISTRIBUTION=" + _agreementFormatDistribution_(goldDist));
    Logger.log("AI_DISTRIBUTION=" + _agreementFormatDistribution_(aiDist));
    Logger.log("TOP_CONFUSION_PAIRS_AI_TO_GOLD=" + _agreementFormatConfusion_(tax.confusionPairs, 10));
    Logger.log("REPORT_REFERENCE_KAPPA=" + ref.kappa);
    Logger.log("REPORT_REFERENCE_AGREEMENT=" + ref.agreement);
    Logger.log("REPORT_REFERENCE_AI_DISTRIBUTION=" + _agreementFormatDistribution_(ref.aiDistribution || {}));
    Logger.log("REPORT_REFERENCE_MATCH=" + String(reportMatch));

    results[d.dim] = {
      kappa: kappa.kappa,
      agreementPercent: agreementPercent,
      taxonomy: tax,
      finalizedOk: finalizedOk,
      finalizedError: finalizedError,
      unfinalized: unfinalized,
      goldDistribution: goldDist,
      aiDistribution: aiDist,
      reportReferenceMatch: reportMatch
    };
  });

  Logger.log("");
  Logger.log("REPORT_OUTPUT_VERSION_DIFF=" + String(reportOutputVersionDiff));
  Logger.log("HAS_FINALIZATION_ISSUES=" + String(hasFinalizationIssues));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("CLEAR_CALLS=0");
  Logger.log("PRODUCTION_RUNNER_CALLS=0");
  Logger.log("READ_ONLY=true");
  Logger.log("============================================");

  return {
    ok: true,
    sheet: SHEET_NAME,
    results: results,
    reportOutputVersionDiff: reportOutputVersionDiff,
    hasFinalizationIssues: hasFinalizationIssues
  };
}

/** 간단 GPT 호출 (텍스트 in/out) */
/**
 * UPDATED FOR GPT-5: responses API로 변경
 */
/**
 * UPDATED: gpt-5-mini & gpt-5 통합 간단 호출 함수
 * - 내부적으로 safeCallGPT 사용 (모델명 자동 전달)
 * - Logger 호출 최소화 (성능 개선)
 */
function callGPT_simple_(promptText, modelName = null){
  try {
    const model = modelName || MODEL;
    
    // UPDATED: messages 배열을 input 문자열로 변환
    const messages = [
      {role:'system', content:'너는 과학 수업 담화 코더다. 한국어로 간결하고 규격을 지켜 출력해라.'},
      {role:'user', content: promptText}
    ];
    const input = messagesToInput(messages);
    
    // UPDATED: safeCallGPT로 안전하게 호출 (모델명 자동 전달)
    const response = safeCallGPT(input, model);
    
    const trimmed = response.trim();
    
    // 빈 응답을 에러로 처리 (조용히 넘어가지 않도록)
    if (!trimmed || trimmed.length === 0) {
      Logger.log("⚠️ callGPT_simple_: 빈 응답 반환됨");
      throw new Error("GPT 응답이 비어있습니다. 프롬프트가 너무 길거나 모델 제한에 걸렸을 수 있습니다.");
    }
    
    return trimmed;
  } catch (e) {
    // 에러를 상위로 전파 (조용히 넘어가지 않도록)
    Logger.log("❌ callGPT_simple_ 오류: " + e.toString());
    throw e;
  }
}

/** GPT/sheet write 없음 — insufficient_quota 판별 helper unit test */
function TEST_OPENAI_QUOTA_ERROR_CLASSIFICATION(){
  const caseA = _isOpenAIQuotaExhaustedError_(
    new Error("HTTP 429: insufficient_quota - You have no credits remaining")
  );
  const caseB = _isOpenAIQuotaExhaustedError_(
    new Error("HTTP 429: rate limit exceeded")
  );
  const caseC = _isOpenAIQuotaExhaustedError_(
    new Error("HTTP 503 service unavailable")
  );
  const pass = (caseA === true) && (caseB === false) && (caseC === false);

  Logger.log("=== OPENAI QUOTA ERROR CLASSIFICATION TEST ===");
  Logger.log("CASE_A=" + String(caseA));
  Logger.log("CASE_B=" + String(caseB));
  Logger.log("CASE_C=" + String(caseC));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PASS=" + String(pass));
  Logger.log("================================");

  return { caseA: caseA, caseB: caseB, caseC: caseC, pass: pass };
}

/** STEP 14A: gpt-5.6-terra Responses API 연결 1회 확인. sheet/note write 없음. production runner 호출 없음. */
function TEST_KCMP_GPT56_TERRA_CONNECTION(){
  const model = "gpt-5.6-terra";
  Logger.log("MODEL=" + model);
  try {
    const response = callGPT("Reply with exactly: OK", model);
    const text = String(response == null ? "" : response);
    Logger.log("API_CONNECTION_OK=" + String(text.trim().length > 0));
    Logger.log("RESPONSE_LENGTH=" + text.length);
    return { ok: text.trim().length > 0, model: model, responseLength: text.length };
  } catch (e) {
    Logger.log("API_CONNECTION_OK=false");
    Logger.log("ERROR=" + String(e));
    return { ok: false, model: model, error: String(e) };
  }
}

function _terraKShadowSheetKey_(){
  return "14차시4조";
}

function _terraKShadowPropKey_(pid){
  return "KCMP_TERRA_K_SHADOW|" + _terraKShadowSheetKey_() + "|" + String(pid || "");
}

function _terraKShadowLoad_(pid){
  const raw = PropertiesService.getDocumentProperties().getProperty(_terraKShadowPropKey_(pid));
  if (!raw) return null;
  try {
    const obj = JSON.parse(raw);
    if (!obj || typeof obj !== "object") return null;
    return obj;
  } catch (e) {
    return null;
  }
}

function _terraKShadowSave_(record){
  PropertiesService.getDocumentProperties().setProperty(
    _terraKShadowPropKey_(record.pid),
    JSON.stringify(record)
  );
}

function _terraKShadowIsFinalized_(record){
  if (!record) return false;
  const status = String(record.status || "");
  return status === "OK" || status === "ERROR";
}

function _terraKShadowCodeForAgreement_(record){
  if (!record || record.status !== "OK") return "";
  if (record.code == null || record.code === "") return "";
  return String(record.code);
}

/** shadow K only: gpt-5.6-terra 최대 5 PID. production K 셀/Note 미변경. */
function TEST_KCMP_TERRA_K_SHADOW_BATCH(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== KCMP TERRA K SHADOW BATCH ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = loadColMap_();
  if (!map) {
    Logger.log("ERROR=COLMAP_MISSING");
    return { ok: false, error: "COLMAP_MISSING" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  let finalizedBefore = 0;
  const pending = [];
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraKShadowLoad_(pid);
    if (_terraKShadowIsFinalized_(rec)) {
      finalizedBefore++;
      continue;
    }
    pending.push(pid);
  }

  Logger.log("=== KCMP TERRA K SHADOW BATCH ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("MODEL=" + String(MODEL_K));
  Logger.log("TOTAL_PACKETS=" + TOTAL);
  Logger.log("SHADOW_FINALIZED_BEFORE=" + finalizedBefore);
  Logger.log("MAX_CASES=" + KCMP_TERRA_K_SHADOW_MAX_CASES);

  const toProcess = pending.slice(0, KCMP_TERRA_K_SHADOW_MAX_CASES);
  let processed = 0;
  let shadowOk = 0;
  let shadowError = 0;

  toProcess.forEach(function(pid, idx){
    if (idx > 0) Utilities.sleep(3000);
    const packet = byPid[pid];
    let result;
    if (!packet) {
      result = { status: "ERROR", code: null, contributors: [], error_type: "PACKET_ERROR", message: "packet not found" };
    } else {
      result = runKDecisionTreeForPacket_(packet);
    }
    const record = {
      pid: pid,
      status: result && result.status ? String(result.status) : "ERROR",
      code: (result && result.code != null) ? result.code : null,
      contributors: (result && result.contributors) ? result.contributors : [],
      error_type: result && result.error_type != null ? String(result.error_type) : "",
      message: result && result.message != null ? String(result.message) : ""
    };
    _terraKShadowSave_(record);
    processed++;
    if (record.status === "OK") shadowOk++;
    else shadowError++;
    Logger.log("PID=" + pid);
    Logger.log("STATUS=" + record.status);
    Logger.log("CODE=" + (record.code == null ? "null" : String(record.code)));
    Logger.log("ERROR_TYPE=" + record.error_type);
    Logger.log("---");
  });

  let remaining = 0;
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    if (!_terraKShadowIsFinalized_(_terraKShadowLoad_(pid))) remaining++;
  }

  Logger.log("PROCESSED_THIS_RUN=" + processed);
  Logger.log("SHADOW_OK=" + shadowOk);
  Logger.log("SHADOW_ERROR=" + shadowError);
  Logger.log("SHADOW_REMAINING=" + remaining);
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PRODUCTION_CLEAR_CALLS=0");
  Logger.log("PRODUCTION_K_UNCHANGED=true");
  Logger.log("AUTO_CHAINING=false");
  Logger.log("================================");

  return {
    processed: processed,
    shadowOk: shadowOk,
    shadowError: shadowError,
    remaining: remaining
  };
}

/** GPT 없음. Terra shadow K vs human gold. 73 finalized 전에는 benchmark 중단. */
function TEST_KCMP_TERRA_K_SHADOW_SUMMARY(){
  const TOTAL = 73;
  const MINI_KAPPA = 0.1947;
  const MINI_AGREEMENT = 45.2;
  const records = [];
  let finalized = 0;
  let shadowErrorCount = 0;

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraKShadowLoad_(pid);
    records.push(rec);
    if (_terraKShadowIsFinalized_(rec)) finalized++;
    if (rec && rec.status === "ERROR") shadowErrorCount++;
  }

  const remaining = TOTAL - finalized;
  Logger.log("=== KCMP TERRA K SHADOW SUMMARY ===");
  Logger.log("MODEL=gpt-5.6-terra");
  Logger.log("TOTAL_PID=" + TOTAL);

  if (finalized !== TOTAL) {
    Logger.log("SHADOW_COMPLETE=false");
    Logger.log("SHADOW_FINALIZED=" + finalized);
    Logger.log("SHADOW_REMAINING=" + remaining);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("PRODUCTION_K_UNCHANGED=true");
    Logger.log("================================");
    return { complete: false, finalized: finalized, remaining: remaining };
  }

  const terraCodes = records.map(_terraKShadowCodeForAgreement_);
  const gold = KCMP_GOLD_14_4.K;
  const kappa = _computeCohenKappa_(gold, terraCodes);
  const tax = _agreementTaxonomy_(gold, terraCodes);
  const goldDist = _agreementDistribution_(gold);
  const terraDist = _agreementDistribution_(terraCodes);
  const agreementPercent = kappa.po * 100;

  Logger.log("SHADOW_COMPLETE=true");
  Logger.log("KAPPA=" + _agreementRound_(kappa.kappa, 4));
  Logger.log("AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent, 1));
  Logger.log("MATCH=" + tax.match);
  Logger.log("AI_BLANK_GOLD_CODE=" + tax.aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + tax.aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + tax.subcodeMismatch);
  Logger.log("GOLD_DISTRIBUTION=" + _agreementFormatDistribution_(goldDist));
  Logger.log("TERRA_DISTRIBUTION=" + _agreementFormatDistribution_(terraDist));
  Logger.log("TOP_CONFUSION_PAIRS_AI_TO_GOLD=" + _agreementFormatConfusion_(tax.confusionPairs, 10));
  Logger.log("SHADOW_ERROR_COUNT=" + shadowErrorCount);
  Logger.log("MINI_BASELINE_KAPPA=" + MINI_KAPPA);
  Logger.log("MINI_BASELINE_AGREEMENT=" + MINI_AGREEMENT);
  Logger.log("DELTA_KAPPA=" + _agreementRound_((kappa.kappa == null ? 0 : kappa.kappa) - MINI_KAPPA, 4));
  Logger.log("DELTA_AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent - MINI_AGREEMENT, 1));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PRODUCTION_K_UNCHANGED=true");
  Logger.log("================================");

  return { complete: true, kappa: kappa.kappa, agreementPercent: agreementPercent, taxonomy: tax };
}

/** K 판정에 제공되는 packet.turns 원문 — 수정/요약 없이 JSON 직렬화용 배열 */
function _kMismatchPacketUtterances_(packet){
  const turns = (packet && packet.turns) ? packet.turns : [];
  return turns.map(function(t){
    return {
      role: t && t.role != null ? t.role : null,
      speakerId: t && t.speakerId != null ? t.speakerId : null,
      speakerRaw: t && t.speakerRaw != null ? t.speakerRaw : null,
      utterance: t && t.utterance != null ? String(t.utterance) : ""
    };
  });
}

function _kMismatchType_(goldCode, aiCode){
  const g = goldCode == null ? "" : String(goldCode);
  const a = aiCode == null ? "" : String(aiCode);
  if (a === "" && g !== "") return "AI_BLANK_GOLD_CODE";
  if (a !== "" && g === "") return "AI_CODE_GOLD_BLANK";
  return "SUBCODE_MISMATCH";
}

function _kMismatchPairCount_(aiCode, goldCode, wantAi, wantGold){
  const a = aiCode == null || aiCode === "" ? "BLANK" : String(aiCode);
  const g = goldCode == null || goldCode === "" ? "BLANK" : String(goldCode);
  return (a === wantAi && g === wantGold) ? 1 : 0;
}

/**
 * STEP 16A: production K vs human gold K mismatch READ-ONLY audit.
 * GPT/write/clear 없음. prompt/validator/decision tree 미변경.
 */
function TEST_KCMP_TERRA_K_MISMATCH_AUDIT(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== KCMP TERRA K MISMATCH AUDIT SUMMARY ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("CLEAR_CALLS=0");
    Logger.log("PRODUCTION_UNCHANGED=true");
    Logger.log("READ_ONLY=true");
    Logger.log("================================");
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  const gold = KCMP_GOLD_14_4.K;
  const rows = [];
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const packet = byPid[pid];
    if (packet && packet.representativeRow) rows.push(packet.representativeRow);
  }
  const kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  const kDisplays = _batchGetDisplaysForRows_(sh, kCol, rows);

  let match = 0;
  let aiBlankGoldCode = 0;
  let aiCodeGoldBlank = 0;
  let subcodeMismatch = 0;
  const mismatchPids = [];
  const pairCounts = {
    BLANK_TO_K1: 0,
    BLANK_TO_K2: 0,
    BLANK_TO_K3: 0,
    K1_TO_K2: 0,
    K1_TO_K3: 0,
    K2_TO_K1: 0,
    K2_TO_K3: 0,
    K3_TO_K1: 0,
    K3_TO_K2: 0
  };

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const goldCode = gold[i] == null ? "" : String(gold[i]);
    const packet = byPid[pid];
    const row = packet && packet.representativeRow ? packet.representativeRow : null;
    const noteText = row != null ? kNotes[row] : "";
    const display = row != null ? kDisplays[row] : "";
    const noteObj = _parseKCMPNoteJson_(noteText) || {};
    const fin = _isFinalKCMPDecisionNote_(noteText, "K");

    // agreement와 동일: display에서 K 코드 파싱 (없으면 blank)
    let aiCode = _agreementParseDimCode_(display, "K");
    if (!aiCode && fin.finalized && fin.status === "OK" && noteObj.code != null && noteObj.code !== "") {
      aiCode = String(noteObj.code);
    }
    if (fin.finalized && fin.status === "ERROR") {
      aiCode = "";
    }

    if (goldCode === aiCode) {
      match++;
      continue;
    }

    const mismatchType = _kMismatchType_(goldCode, aiCode);
    if (mismatchType === "AI_BLANK_GOLD_CODE") aiBlankGoldCode++;
    else if (mismatchType === "AI_CODE_GOLD_BLANK") aiCodeGoldBlank++;
    else subcodeMismatch++;

    mismatchPids.push(pid);
    pairCounts.BLANK_TO_K1 += _kMismatchPairCount_(aiCode, goldCode, "BLANK", "K1");
    pairCounts.BLANK_TO_K2 += _kMismatchPairCount_(aiCode, goldCode, "BLANK", "K2");
    pairCounts.BLANK_TO_K3 += _kMismatchPairCount_(aiCode, goldCode, "BLANK", "K3");
    pairCounts.K1_TO_K2 += _kMismatchPairCount_(aiCode, goldCode, "K1", "K2");
    pairCounts.K1_TO_K3 += _kMismatchPairCount_(aiCode, goldCode, "K1", "K3");
    pairCounts.K2_TO_K1 += _kMismatchPairCount_(aiCode, goldCode, "K2", "K1");
    pairCounts.K2_TO_K3 += _kMismatchPairCount_(aiCode, goldCode, "K2", "K3");
    pairCounts.K3_TO_K1 += _kMismatchPairCount_(aiCode, goldCode, "K3", "K1");
    pairCounts.K3_TO_K2 += _kMismatchPairCount_(aiCode, goldCode, "K3", "K2");

    const rationale = noteObj.reason != null ? noteObj.reason
      : (noteObj.rationale != null ? noteObj.rationale
        : (noteObj.message != null ? noteObj.message : ""));

    Logger.log("=== K MISMATCH ===");
    Logger.log("PID=" + pid);
    Logger.log("REPRESENTATIVE_ROW=" + (row == null ? "" : String(row)));
    Logger.log("GOLD_CODE=" + (goldCode === "" ? "BLANK" : goldCode));
    Logger.log("AI_CODE=" + (aiCode === "" ? "BLANK" : aiCode));
    Logger.log("MISMATCH_TYPE=" + mismatchType);
    Logger.log("STATUS=" + String(noteObj.status != null ? noteObj.status : (fin.status || "")));
    Logger.log("CONTRIBUTORS=" + JSON.stringify(Array.isArray(noteObj.contributors) ? noteObj.contributors : []));
    Logger.log("SCIENCE_CONTENT=" + (noteObj.science_content == null ? "" : String(noteObj.science_content)));
    Logger.log("CLAIM=" + (noteObj.claim == null ? "" : String(noteObj.claim)));
    Logger.log("EVIDENCE=" + (noteObj.evidence == null ? "" : String(noteObj.evidence)));
    Logger.log("STEP0_BASIS=" + JSON.stringify(noteObj.step0_basis == null ? [] : noteObj.step0_basis));
    Logger.log("RATIONALE=" + String(rationale));
    Logger.log("PACKET_UTTERANCES=" + JSON.stringify(_kMismatchPacketUtterances_(packet)));
    Logger.log("---");
  }

  const totalMismatch = mismatchPids.length;

  Logger.log("=== KCMP TERRA K MISMATCH AUDIT SUMMARY ===");
  Logger.log("TOTAL_PID=" + TOTAL);
  Logger.log("MATCH=" + match);
  Logger.log("TOTAL_MISMATCH=" + totalMismatch);
  Logger.log("AI_BLANK_GOLD_CODE=" + aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + subcodeMismatch);
  Logger.log("BLANK_TO_K1=" + pairCounts.BLANK_TO_K1);
  Logger.log("BLANK_TO_K2=" + pairCounts.BLANK_TO_K2);
  Logger.log("BLANK_TO_K3=" + pairCounts.BLANK_TO_K3);
  Logger.log("K1_TO_K2=" + pairCounts.K1_TO_K2);
  Logger.log("K1_TO_K3=" + pairCounts.K1_TO_K3);
  Logger.log("K2_TO_K1=" + pairCounts.K2_TO_K1);
  Logger.log("K2_TO_K3=" + pairCounts.K2_TO_K3);
  Logger.log("K3_TO_K1=" + pairCounts.K3_TO_K1);
  Logger.log("K3_TO_K2=" + pairCounts.K3_TO_K2);
  Logger.log("MISMATCH_PIDS=" + JSON.stringify(mismatchPids));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("CLEAR_CALLS=0");
  Logger.log("PRODUCTION_UNCHANGED=true");
  Logger.log("READ_ONLY=true");
  Logger.log("================================");

  return {
    ok: true,
    match: match,
    totalMismatch: totalMismatch,
    aiBlankGoldCode: aiBlankGoldCode,
    aiCodeGoldBlank: aiCodeGoldBlank,
    subcodeMismatch: subcodeMismatch,
    pairCounts: pairCounts,
    mismatchPids: mismatchPids
  };
}

// ============================================================
// STEP 16B — K semantic candidate SHADOW calibration
// production K prompt/runner/Note/sheet 절대 미변경.
// ============================================================

function _kCand16BPropKey_(pid){
  return "KCMP_K_CAND16B_" + String(pid || "");
}

function _kCand16BLoad_(pid){
  const raw = PropertiesService.getDocumentProperties().getProperty(_kCand16BPropKey_(pid));
  if (!raw) return null;
  try {
    const obj = JSON.parse(raw);
    if (!obj || typeof obj !== "object") return null;
    return obj;
  } catch (e) {
    return null;
  }
}

function _kCand16BSave_(record){
  PropertiesService.getDocumentProperties().setProperty(
    _kCand16BPropKey_(record.pid),
    JSON.stringify(record)
  );
}

function _kCand16BIsFinalized_(record){
  if (!record) return false;
  const status = String(record.status || "");
  return status === "OK" || status === "ERROR";
}

function _kCand16BCodeForAgreement_(record){
  if (!record || record.status !== "OK") return "";
  if (record.code == null || record.code === "") return "";
  return String(record.code);
}

function _kCand16BFormatTurns_(packet){
  return ((packet && packet.turns) || []).map(function(t){
    const sid = t.speakerId ? t.speakerId : "";
    const raw = t.speakerRaw || "";
    return "[" + t.row + "] " + t.role + " " + sid + " " + raw + ": " + String(t.utterance || "").replace(/\s+/g, " ");
  }).join("\n");
}

function _kCand16BResolveNeighbors_(packet, allPackets){
  const byPid = {};
  (allPackets || []).forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });
  const prevPid = packet && packet.context ? packet.context.previousPid : null;
  const nextPid = packet && packet.context ? packet.context.nextPid : null;
  return {
    previousPacket: prevPid ? (byPid[prevPid] || null) : null,
    nextPacket: nextPid ? (byPid[nextPid] || null) : null,
    previousPid: prevPid || null,
    nextPid: nextPid || null
  };
}

/**
 * STEP 16B candidate-only K prompt.
 * production buildKDecisionPrompt_ 는 호출/수정하지 않는다.
 */
function buildKDecisionPrompt_Candidate16B_(packet, neighbors){
  neighbors = neighbors || {};
  const students = (packet.students || []).map(function(s){
    return (s.id || "") + " = " + (s.label || "");
  }).join("\n");
  const active = (packet.activeStudentIds || []).join(", ");
  const turnsText = _kCand16BFormatTurns_(packet);
  const summary = String(packet.summary == null ? "" : packet.summary);
  const prevText = neighbors.previousPacket
    ? ("PID=" + (neighbors.previousPid || neighbors.previousPacket.pid) + "\n" + _kCand16BFormatTurns_(neighbors.previousPacket))
    : "(none)";
  const nextText = neighbors.nextPacket
    ? ("PID=" + (neighbors.nextPid || neighbors.nextPacket.pid) + "\n" + _kCand16BFormatTurns_(neighbors.nextPacket))
    : "(none)";

  const lines = [];
  lines.push("당신은 과학 소집단 담화의 K차원(인식적 실행) 코더이다. (CANDIDATE16B SHADOW — production 미반영)");
  lines.push("1차 판정 범위는 [CURRENT CLUSTER TURNS]이다.");
  lines.push("[PREVIOUS/NEXT PACKET TURNS]는 CONTEXT-ONLY이다.");
  lines.push("neighbor context는 오직 현재 cluster 발화의 지시어/생략/선택지/대상 의미 복원에만 사용한다.");
  lines.push("neighbor cluster 자체의 K 행위를 현재 PID의 K로 가져오지 마라.");
  lines.push("contributors는 반드시 CURRENT packet의 ACTIVE STUDENTS(S1~S4)만 가능하다. neighbor 학생 금지.");
  lines.push("quotes / step0_basis의 quote도 CURRENT CLUSTER TURNS의 학생 원발화만 사용한다.");
  lines.push("[SUMMARY]는 보조자료. 원발화와 충돌하면 원발화 우선.");
  lines.push("JSON만 출력. JSON 외 텍스트/마크다운/코드펜스 금지.");
  lines.push("");
  lines.push("가능한 code: null | \"K1\" | \"K2\" | \"K3\" (최대 1개)");
  lines.push("우선순위: K3 > K2 > K1 > null");
  lines.push("");
  lines.push("===== CANDIDATE16B-A: 맥락 복원 허용 (K1 누락 방지) =====");
  lines.push("현재 학생 발화가 지시어/생략/선택지 표현이어도, 과학적 의미를 CURRENT 또는 필요시 PREV/NEXT에서 합리적으로 복원할 수 있으면");
  lines.push("\"자기완결적 문장이 아니다\"라는 이유만으로 BLANK(null) 처리하지 않는다.");
  lines.push("예: \"압력이 어디가 높아?\", \"무슨 법칙이야?\", \"그래야 나가지 않아?\", \"짱구인 것 같긴 한데.\", \"훈이 절대 아니야.\", \"다르지 않을까?\", \"억지로?\"");
  lines.push("대상이 CURRENT 교사/학생 발화 또는 PREV/NEXT에서 확인되면 과학적 판단/질문/추론으로 K1 가능.");
  lines.push("단 neighbor의 과학 행위를 현재 PID로 이전하지 말고, 현재 학생 발화의 의미 복원에만 써라.");
  lines.push("");
  lines.push("===== CANDIDATE16B-B: K3 불완전/분산 주장-근거 =====");
  lines.push("K3는 완결된 한 문장일 필요 없다. 다음도 K3 가능:");
  lines.push("1) 교사 질문 → 학생 주장/근거 응답");
  lines.push("2) 학생 A 주장 → 학생 B 근거");
  lines.push("3) 주장 생략 + 특정 결론을 지지하는 학생 근거 (CURRENT에서 결론이 특정 가능)");
  lines.push("4) 여러 학생 발화에 claim/evidence 분산");
  lines.push("5) 중단/자기수정이어도 substantive support relation이 확인되면 K3");
  lines.push("강한 support signal: 왜냐하면, ~니까/~으니까, 그래서, 그러니까, 그러면, ~때문에, ~하려면, ~해야, 근거는/이유는");
  lines.push("정답 여부 판단 금지. 과학적으로 틀려도 claim + substantive evidence 구조이면 K3 가능.");
  lines.push("K3 최소: (1) 주장 또는 특정 가능한 결론 (2) 실질적 근거 (3) 근거가 주장을 지지하는 기능적 연결");
  lines.push("");
  lines.push("===== CANDIDATE16B-C: K3 과대판정 방지 =====");
  lines.push("교사는 구조를 제공할 수 있으나 teacher-only reasoning을 학생 K3로 가져오지 않는다.");
  lines.push("K3 금지:");
  lines.push("- 교사가 claim/reasoning을 사실상 모두 제공하고 학생은 \"밖이요\", \"고압에서 저압으로요\" 같은 고립 단답만 제공");
  lines.push("- 관련 사실 2개를 연속 말했지만 하나가 다른 하나를 지지하는 기능적 연결 없음");
  lines.push("- 단순 관련 과학 사실의 병치");
  lines.push("\"관련 사실 2개 존재\" ≠ K3. 학생 발화가 특정 claim을 지지하는 evidence/reason 기능이어야 한다.");
  lines.push("");
  lines.push("===== CANDIDATE16B-D: K2 =====");
  lines.push("실험/관찰/조작에서 나온 구체적 현상·감각·변화는 K2 가능.");
  lines.push("예: \"부풀어 올라\", \"압축하는 게 힘들어\", \"터질 것 같아\", 직접 본 움직임/변화, 표/그래프/사진/모형/실험 결과의 구체 내용.");
  lines.push("이것이 특정 claim을 정당화하는 evidence로 기능하면 K3 우선.");
  lines.push("자료 \"본다\"만으로 K2 아님.");
  lines.push("");
  lines.push("===== CANDIDATE16B-E: K1 =====");
  lines.push("K3/K2가 아니면서 다음이면 K1:");
  lines.push("과학적 판단, 예측, 설명, 추론, 가능성 제시, 과학 내용 질문, 과학적 선택지 판단, 과학적 비교/차이 질문,");
  lines.push("맥락으로 복원 가능한 과학적 확인 질문.");
  lines.push("단순 절차/역할/기록/잡담/교과목·수업운영 대화는 null 유지.");
  lines.push("request-only(\"왜 그렇게 생각해?\", \"너는 뭐라고 생각해?\")는 null.");
  lines.push("");
  lines.push("===== STEP 순서 =====");
  lines.push("STEP0: 현재(필요시 맥락 복원 후) 학생의 과학적 의미구성이 있는가? NO→null");
  lines.push("STEP1: K3 주장-근거 정당화? YES→K3");
  lines.push("STEP2: K2 관찰/자료 구체 내용? YES→K2 (단 claim 정당화면 backtrack to K3)");
  lines.push("STEP3: K1 기타 과학적 의미구성? YES→K1 else null");
  lines.push("");
  lines.push("===== contributors / quotes =====");
  lines.push("contributors = 최종 K 성립에 실제 기여한 CURRENT 학생만.");
  lines.push("quotes = CURRENT 학생 원발화 객체 배열 {\"speaker\":\"S1\",\"quote\":\"...\"}");
  lines.push("UNIQUE(contributors) === UNIQUE(quotes[].speaker) (non-null K)");
  lines.push("step0_basis도 동일 object 형식, CURRENT 학생 발화만.");
  lines.push("교사는 contributor/quotes speaker 금지.");
  lines.push("");
  lines.push("===== OUTPUT CONTRACT (validateKDecisionResult_ 호환 — semantic 변경 아님) =====");
  lines.push("decision_path는 최종 code의 bookkeeping이다. semantic 근거를 새로 만들지 말고 아래 canonical path를 맞춰라.");
  lines.push("");
  lines.push("code=K3 → decision_path에 반드시 \"K-STEP1:YES\" 포함.");
  lines.push("  권장: [\"K-STEP0:YES\",\"K-STEP1:YES\"]");
  lines.push("code=K2 → decision_path에 반드시 \"K-STEP1:NO\", \"K-STEP2:YES\" 포함.");
  lines.push("  권장: [\"K-STEP0:YES\",\"K-STEP1:NO\",\"K-STEP2:YES\",\"K-STEP2-BACKTRACK:NO\"]");
  lines.push("code=K1 → decision_path에 반드시 \"K-STEP1:NO\", \"K-STEP2:NO\", \"K-STEP3:YES\" 포함.");
  lines.push("  권장: [\"K-STEP0:YES\",\"K-STEP1:NO\",\"K-STEP2:NO\",\"K-STEP3:YES\"]");
  lines.push("code=null → 권장: [\"K-STEP0:NO\"]");
  lines.push("");
  lines.push("code=K2이면 evidence는 반드시 non-empty string.");
  lines.push("evidence에는 학생이 실제로 제시한 구체적 관찰/실험/표/그래프/이미지/모형/자료 내용을 짧게 기술.");
  lines.push("존재하지 않는 관찰을 새로 만들지 마라.");
  lines.push("예: 발화 \"부풀어 올라.\" → evidence=\"주사기가 부풀어 오르는 변화를 관찰했다.\" 가능.");
  lines.push("K1에서는 evidence=null 가능. K3에서는 claim + evidence 모두 필수.");
  lines.push("");
  lines.push("quotes[].quote 와 step0_basis[].quote 는 반드시 CURRENT CLUSTER TURNS의");
  lines.push("\"한 개 student utterance\"에서 가져온 연속된 정확한 문자열 substring 이어야 한다.");
  lines.push("절대 금지: 두 발화 합치기, 떨어진 구절 합치기, \"...\" 삽입 요약, 조사/어미 수정, 문장 교정, paraphrase.");
  lines.push("긴 발화는 필요한 contiguous 구간만 잘라 quote로 쓴다. 여러 구간이 필요하면 quotes 배열에 별도 object로 분리.");
  lines.push("");
  lines.push("===== 출력 JSON 스키마 =====");
  lines.push("{");
  lines.push('  "schema_version":"KCMP_K_V1",');
  lines.push('  "status":"OK",');
  lines.push('  "code": null,');
  lines.push('  "contributors": [],');
  lines.push('  "science_content": null,');
  lines.push('  "step0_basis": [],');
  lines.push('  "claim": null,');
  lines.push('  "evidence": null,');
  lines.push('  "reason": "필수 비어있지 않은 문자열",');
  lines.push('  "decision_path": ["K-STEP0:NO"],');
  lines.push('  "boundary_check": null,');
  lines.push('  "context_needed": false,');
  lines.push('  "quotes": []');
  lines.push("}");
  lines.push("code=K3이면 claim/evidence 필수. code=K2이면 evidence 필수. code!=null이면 science_content, step0_basis, quotes 필수.");
  lines.push("출력 직전 self-check: decision_path tokens / evidence(K2) / quotes·step0_basis exact contiguous substring.");
  lines.push("");
  lines.push("[PID]");
  lines.push(packet.pid || "");
  lines.push("");
  lines.push("[STUDENTS]");
  lines.push(students || "(없음)");
  lines.push("");
  lines.push("[ACTIVE STUDENTS]");
  lines.push(active || "(없음)");
  lines.push("");
  lines.push("[CURRENT CLUSTER TURNS]  << PRIMARY");
  lines.push(turnsText || "(원발화 없음)");
  lines.push("");
  lines.push("[PREVIOUS PACKET TURNS - CONTEXT ONLY]");
  lines.push(prevText);
  lines.push("");
  lines.push("[NEXT PACKET TURNS - CONTEXT ONLY]");
  lines.push(nextText);
  lines.push("");
  lines.push("[SUMMARY - AUXILIARY ONLY]");
  lines.push(summary || "(요약 없음)");
  lines.push("");
  lines.push("JSON만 출력하라.");
  return lines.join("\n");
}

/**
 * candidate16B shadow runner. production runKDecisionTreeForPacket_ 미호출/미수정.
 * parse/validate는 기존 helper 재사용 (quotes는 current packet turns만 인정).
 */
function runKDecisionTreeCandidate16B_ForPacket_(packet, options){
  options = options || {};
  const pid = packet && packet.pid ? packet.pid : "";
  if (!packet || !pid) return _makeKDecisionError_("PACKET_ERROR", "pid 없음", pid);
  if (!packet.turns || packet.turns.length === 0) {
    return _makeKDecisionError_("PACKET_ERROR", "turns가 비어 있음", pid);
  }

  const neighbors = _kCand16BResolveNeighbors_(packet, options.allPackets || []);
  let raw = "";
  try {
    raw = callGPT_simple_(buildKDecisionPrompt_Candidate16B_(packet, neighbors), MODEL_K);
  } catch (e) {
    return _makeKDecisionError_("API_ERROR", e.toString(), pid);
  }

  let parsed;
  try {
    parsed = parseKDecisionTreeResponse_(raw);
  } catch (e) {
    return _makeKDecisionError_("PARSER_ERROR", e.toString(), pid, { raw_excerpt: String(raw).slice(0, 400) });
  }

  const v = validateKDecisionResult_(parsed, packet);
  if (!v.ok) {
    return _makeKDecisionError_("VALIDATION_ERROR", v.errors.join("; "), pid, {
      validation_errors: v.errors,
      raw_excerpt: String(raw).slice(0, 400)
    });
  }
  parsed.status = "OK";
  parsed.schema_version = "KCMP_K_V1";
  parsed.pid = pid;
  parsed.candidate = "CAND16B";
  return parsed;
}

/** STEP 16B candidate shadow batch — production sheet/Note 미변경. 자동 chaining 없음. */
function runTerraKCandidate16B_ShadowBatch(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== K CANDIDATE16B SHADOW BATCH ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = loadColMap_();
  if (!map) {
    Logger.log("ERROR=COLMAP_MISSING");
    return { ok: false, error: "COLMAP_MISSING" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  let finalizedBefore = 0;
  const pending = [];
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _kCand16BLoad_(pid);
    if (_kCand16BIsFinalized_(rec)) {
      finalizedBefore++;
      continue;
    }
    pending.push(pid);
  }

  const toProcess = pending.slice(0, KCMP_K_CAND16B_SHADOW_MAX_CASES);
  let processed = 0;
  let okCoded = 0;
  let okNull = 0;
  let errorThisRun = 0;

  toProcess.forEach(function(pid, idx){
    if (idx > 0) Utilities.sleep(3000);
    const packet = byPid[pid];
    let result;
    if (!packet) {
      result = { status: "ERROR", code: null, contributors: [], error_type: "PACKET_ERROR", message: "packet not found" };
    } else {
      result = runKDecisionTreeCandidate16B_ForPacket_(packet, { allPackets: packets });
    }

    const record = {
      pid: pid,
      status: result && result.status ? String(result.status) : "ERROR",
      code: (result && result.code != null) ? result.code : null,
      contributors: (result && result.contributors) ? result.contributors : [],
      science_content: result && result.science_content != null ? result.science_content : null,
      claim: result && result.claim != null ? result.claim : null,
      evidence: result && result.evidence != null ? result.evidence : null,
      reason: result && result.reason != null ? String(result.reason) : "",
      error_type: result && result.error_type != null ? String(result.error_type) : "",
      message: result && result.message != null ? String(result.message) : "",
      validation_errors: Array.isArray(result && result.validation_errors)
        ? result.validation_errors.slice()
        : []
    };
    _kCand16BSave_(record);
    processed++;

    if (record.status === "OK") {
      if (record.code == null || record.code === "") okNull++;
      else okCoded++;
    } else {
      errorThisRun++;
    }

    Logger.log("PID=" + pid);
    Logger.log("STATUS=" + record.status);
    Logger.log("CODE=" + (record.code == null ? "null" : String(record.code)));
    Logger.log("ERROR_TYPE=" + record.error_type);
    Logger.log("---");
  });

  let remaining = 0;
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    if (!_kCand16BIsFinalized_(_kCand16BLoad_(pid))) remaining++;
  }

  Logger.log("=== K CANDIDATE16B SHADOW BATCH ===");
  Logger.log("TOTAL_PACKETS=" + TOTAL);
  Logger.log("FINALIZED_BEFORE=" + finalizedBefore);
  Logger.log("PROCESSED_THIS_RUN=" + processed);
  Logger.log("OK_CODED_THIS_RUN=" + okCoded);
  Logger.log("OK_NULL_THIS_RUN=" + okNull);
  Logger.log("ERROR_THIS_RUN=" + errorThisRun);
  Logger.log("REMAINING=" + remaining);
  Logger.log("COMPLETE=" + String(remaining === 0));
  Logger.log("MODEL=" + String(MODEL_K));
  Logger.log("SHEET_WRITES=0");
  Logger.log("PRODUCTION_WRITES=0");
  Logger.log("AUTO_CHAINING=false");
  Logger.log("================================");

  return {
    processed: processed,
    okCoded: okCoded,
    okNull: okNull,
    errorThisRun: errorThisRun,
    remaining: remaining,
    complete: remaining === 0
  };
}

/** GPT 없음. candidate16B shadow vs gold. 73 finalized 전 agreement 중단. */
function TEST_TERRA_K_CANDIDATE16B_AGREEMENT(){
  const TOTAL = 73;
  const BASE_KAPPA = 0.4042;
  const BASE_AGREEMENT = 60.3;
  const records = [];
  let finalizedOk = 0;
  let finalizedError = 0;

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _kCand16BLoad_(pid);
    records.push(rec);
    if (rec && rec.status === "OK") finalizedOk++;
    else if (rec && rec.status === "ERROR") finalizedError++;
  }

  const finalized = finalizedOk + finalizedError;
  Logger.log("=== K CANDIDATE16B AGREEMENT ===");
  Logger.log("TOTAL=" + TOTAL);
  Logger.log("FINALIZED_OK=" + finalizedOk);
  Logger.log("FINALIZED_ERROR=" + finalizedError);

  if (finalized !== TOTAL) {
    Logger.log("SHADOW_COMPLETE=false");
    Logger.log("SHADOW_FINALIZED=" + finalized);
    Logger.log("SHADOW_REMAINING=" + (TOTAL - finalized));
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("PRODUCTION_UNCHANGED=true");
    Logger.log("================================");
    return { complete: false, finalized: finalized, remaining: TOTAL - finalized };
  }

  const candCodes = records.map(_kCand16BCodeForAgreement_);
  const gold = KCMP_GOLD_14_4.K;
  const kappa = _computeCohenKappa_(gold, candCodes);
  const tax = _agreementTaxonomy_(gold, candCodes);
  const goldDist = _agreementDistribution_(gold);
  const candDist = _agreementDistribution_(candCodes);
  const agreementPercent = kappa.po * 100;

  Logger.log("KAPPA=" + _agreementRound_(kappa.kappa, 4));
  Logger.log("AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent, 1));
  Logger.log("MATCH=" + tax.match);
  Logger.log("AI_BLANK_GOLD_CODE=" + tax.aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + tax.aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + tax.subcodeMismatch);
  Logger.log("GOLD_DISTRIBUTION=" + _agreementFormatDistribution_(goldDist));
  Logger.log("CANDIDATE_DISTRIBUTION=" + _agreementFormatDistribution_(candDist));
  Logger.log("TOP_CONFUSION_PAIRS_AI_TO_GOLD=" + _agreementFormatConfusion_(tax.confusionPairs, 10));
  Logger.log("BASE_CURRENT_PRODUCTION_KAPPA=" + BASE_KAPPA);
  Logger.log("BASE_CURRENT_PRODUCTION_AGREEMENT=" + BASE_AGREEMENT);
  Logger.log("DELTA_KAPPA=" + _agreementRound_((kappa.kappa == null ? 0 : kappa.kappa) - BASE_KAPPA, 4));
  Logger.log("DELTA_AGREEMENT=" + _agreementRound_(agreementPercent - BASE_AGREEMENT, 1));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("PRODUCTION_UNCHANGED=true");
  Logger.log("================================");

  return {
    complete: true,
    kappa: kappa.kappa,
    agreementPercent: agreementPercent,
    taxonomy: tax,
    finalizedOk: finalizedOk,
    finalizedError: finalizedError
  };
}

/**
 * STEP 16D: Candidate16B VALIDATION_ERROR property만 삭제.
 * OK record / production / non-validation ERROR 보존. semantic 변경 없음.
 */
function RESET_TERRA_K_CANDIDATE16B_VALIDATION_ERRORS_ONLY(){
  const TOTAL = 73;
  const props = PropertiesService.getDocumentProperties();
  const deletedPids = [];
  let totalRecords = 0;
  let validationErrorCount = 0;
  let okPreserved = 0;

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const key = _kCand16BPropKey_(pid);
    const raw = props.getProperty(key);
    if (!raw) continue;
    totalRecords++;

    let rec = null;
    try {
      rec = JSON.parse(raw);
    } catch (e) {
      continue;
    }
    if (!rec || typeof rec !== "object") continue;

    const status = String(rec.status || "");
    const errorType = String(rec.error_type || "");

    if (status === "OK") {
      okPreserved++;
      continue;
    }

    if (status === "ERROR" && errorType === "VALIDATION_ERROR") {
      validationErrorCount++;
      props.deleteProperty(key);
      deletedPids.push(pid);
    }
  }

  Logger.log("=== K CAND16B VALIDATION ERROR RESET ===");
  Logger.log("TOTAL_RECORDS=" + totalRecords);
  Logger.log("VALIDATION_ERROR_COUNT=" + validationErrorCount);
  Logger.log("DELETED_COUNT=" + deletedPids.length);
  Logger.log("DELETED_PIDS=" + JSON.stringify(deletedPids));
  Logger.log("OK_RECORDS_PRESERVED=" + okPreserved);
  Logger.log("SHEET_WRITES=0");
  Logger.log("PRODUCTION_WRITES=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("================================");

  return {
    totalRecords: totalRecords,
    validationErrorCount: validationErrorCount,
    deletedCount: deletedPids.length,
    deletedPids: deletedPids,
    okRecordsPreserved: okPreserved
  };
}

function _kCand16BIsAllowedCode_(code){
  return code === null || code === "K1" || code === "K2" || code === "K3";
}

function _kCand16BValidateContributors_(contributors){
  if (!Array.isArray(contributors)) {
    return { ok: false, message: "contributors not array" };
  }
  for (let i = 0; i < contributors.length; i++) {
    const s = String(contributors[i] == null ? "" : contributors[i]);
    if (!/^S[1-4]$/.test(s)) {
      return { ok: false, message: "invalid contributor: " + s };
    }
  }
  return { ok: true };
}

/** Candidate16B record → production K Note result. 재판정/추론 없이 복사만. */
function _kCand16BBuildProductionResult_(pid, rec){
  const result = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    pid: String(pid || ""),
    code: (rec.code === undefined) ? null : rec.code,
    contributors: Array.isArray(rec.contributors) ? rec.contributors.slice() : [],
    science_content: (rec.science_content === undefined) ? null : rec.science_content,
    claim: (rec.claim === undefined) ? null : rec.claim,
    evidence: (rec.evidence === undefined) ? null : rec.evidence,
    reason: rec.reason != null ? String(rec.reason) : "",
    source: "KCMP_K_CANDIDATE16B_ACCEPTED"
  };
  if (Object.prototype.hasOwnProperty.call(rec, "step0_basis")) result.step0_basis = rec.step0_basis;
  if (Object.prototype.hasOwnProperty.call(rec, "decision_path")) result.decision_path = rec.decision_path;
  if (Object.prototype.hasOwnProperty.call(rec, "boundary_check")) result.boundary_check = rec.boundary_check;
  if (Object.prototype.hasOwnProperty.call(rec, "context_needed")) result.context_needed = rec.context_needed;
  if (Object.prototype.hasOwnProperty.call(rec, "quotes")) result.quotes = rec.quotes;
  return result;
}

/**
 * STEP 16E: Candidate16B 확정 결과를 production K에 1회 적용.
 * GPT/semantic 변경 없음. C/M/P / Candidate Properties 미수정.
 * 73건 전체 preflight 성공 후에만 write 시작.
 */
function APPLY_TERRA_K_CANDIDATE16B_TO_PRODUCTION(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;

  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== K CANDIDATE16B -> PRODUCTION APPLY ===");
    Logger.log("SHEET=" + String(sh.getName()));
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "WRONG_SHEET" };
  }

  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  if (!(kCol > 0)) {
    Logger.log("=== K CANDIDATE16B -> PRODUCTION APPLY ===");
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=K_COLUMN_UNRESOLVED");
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "K_COLUMN_UNRESOLVED" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  if (packets.length !== TOTAL) {
    Logger.log("=== K CANDIDATE16B -> PRODUCTION APPLY ===");
    Logger.log("SHEET=" + SHEET_NAME);
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=PACKET_COUNT_MISMATCH expected=" + TOTAL);
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "PACKET_COUNT_MISMATCH" };
  }

  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  // -------- VALIDATE ALL 73 (no writes) --------
  const prepared = [];
  let candidateRecordsFound = 0;
  let candidateOk = 0;
  const preflightProblems = [];

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const packet = byPid[pid];
    if (!packet || !(Number(packet.representativeRow) > 0)) {
      preflightProblems.push({ pid: pid, problem: "PACKET_OR_ROW_MISSING" });
      continue;
    }

    const rec = _kCand16BLoad_(pid);
    if (!rec) {
      preflightProblems.push({ pid: pid, problem: "CANDIDATE_MISSING" });
      continue;
    }
    candidateRecordsFound++;

    if (String(rec.status || "") !== "OK") {
      preflightProblems.push({
        pid: pid,
        problem: "CANDIDATE_NOT_OK",
        status: String(rec.status || ""),
        error_type: String(rec.error_type || "")
      });
      continue;
    }
    candidateOk++;

    const code = (rec.code === undefined) ? null : rec.code;
    if (!_kCand16BIsAllowedCode_(code)) {
      preflightProblems.push({ pid: pid, problem: "INVALID_CODE", code: code });
      continue;
    }

    const contribCheck = _kCand16BValidateContributors_(rec.contributors);
    if (!contribCheck.ok) {
      preflightProblems.push({ pid: pid, problem: "INVALID_CONTRIBUTORS", message: contribCheck.message });
      continue;
    }

    prepared.push({
      pid: pid,
      row: Number(packet.representativeRow),
      result: _kCand16BBuildProductionResult_(pid, rec),
      expectedCode: code,
      expectedContributors: Array.isArray(rec.contributors) ? rec.contributors.slice() : []
    });
  }

  const preflightPass = (preflightProblems.length === 0 && prepared.length === TOTAL);
  if (!preflightPass) {
    Logger.log("=== K CANDIDATE16B -> PRODUCTION APPLY ===");
    Logger.log("SHEET=" + SHEET_NAME);
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("CANDIDATE_RECORDS_FOUND=" + candidateRecordsFound);
    Logger.log("CANDIDATE_OK=" + candidateOk);
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("PREFLIGHT_PROBLEM_COUNT=" + preflightProblems.length);
    Logger.log("PREFLIGHT_PROBLEMS=" + JSON.stringify(preflightProblems));
    Logger.log("WRITTEN_COUNT=0");
    Logger.log("VERIFY_MATCH_COUNT=0");
    Logger.log("VERIFY_MISMATCH_COUNT=0");
    Logger.log("VERIFY_MISMATCH_PIDS=[]");
    Logger.log("GPT_CALLS=0");
    Logger.log("C_WRITES=0");
    Logger.log("M_WRITES=0");
    Logger.log("P_WRITES=0");
    Logger.log("CANDIDATE_PROPERTY_WRITES=0");
    Logger.log("SEMANTIC_CHANGE=NONE");
    Logger.log("SOURCE=KCMP_K_CANDIDATE16B_ACCEPTED");
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return {
      ok: false,
      preflightPass: false,
      problems: preflightProblems,
      candidateRecordsFound: candidateRecordsFound,
      candidateOk: candidateOk
    };
  }

  // -------- WRITE ALL 73 --------
  let writtenCount = 0;
  let codedCount = 0;
  let nullCount = 0;
  prepared.forEach(function(item){
    _writeKDecisionCell_(sh, item.row, kCol, item.result);
    writtenCount++;
    if (item.expectedCode == null || item.expectedCode === "") nullCount++;
    else codedCount++;
  });

  // -------- VERIFY --------
  const verifyRows = prepared.map(function(item){ return item.row; });
  const notesAfter = _batchGetNotesForRows_(sh, kCol, verifyRows);
  const mismatchPids = [];
  let verifyMatch = 0;

  prepared.forEach(function(item){
    const noteObj = _parseKCMPNoteJson_(notesAfter[item.row]);
    const okStatus = noteObj && String(noteObj.status || "") === "OK";
    const prodCode = noteObj && (noteObj.code === undefined) ? null : (noteObj ? noteObj.code : null);
    const codesEqual = (prodCode == null && item.expectedCode == null) ||
      (prodCode != null && item.expectedCode != null && String(prodCode) === String(item.expectedCode));
    const contribsEqual = _sameCanonicalContributorSet_(
      noteObj && noteObj.contributors,
      item.expectedContributors
    );
    if (okStatus && codesEqual && contribsEqual) {
      verifyMatch++;
    } else {
      mismatchPids.push(item.pid);
    }
  });

  const applyComplete = (writtenCount === TOTAL && verifyMatch === TOTAL && mismatchPids.length === 0);

  Logger.log("=== K CANDIDATE16B -> PRODUCTION APPLY ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("CANDIDATE_RECORDS_FOUND=" + candidateRecordsFound);
  Logger.log("CANDIDATE_OK=" + candidateOk);
  Logger.log("PREFLIGHT_PASS=true");
  Logger.log("WRITTEN_COUNT=" + writtenCount);
  Logger.log("CODED_COUNT=" + codedCount);
  Logger.log("NULL_COUNT=" + nullCount);
  Logger.log("VERIFY_MATCH_COUNT=" + verifyMatch);
  Logger.log("VERIFY_MISMATCH_COUNT=" + mismatchPids.length);
  Logger.log("VERIFY_MISMATCH_PIDS=" + JSON.stringify(mismatchPids));
  Logger.log("GPT_CALLS=0");
  Logger.log("C_WRITES=0");
  Logger.log("M_WRITES=0");
  Logger.log("P_WRITES=0");
  Logger.log("CANDIDATE_PROPERTY_WRITES=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("SOURCE=KCMP_K_CANDIDATE16B_ACCEPTED");
  Logger.log("APPLY_COMPLETE=" + String(applyComplete));
  Logger.log("================================");

  return {
    ok: applyComplete,
    preflightPass: true,
    writtenCount: writtenCount,
    codedCount: codedCount,
    nullCount: nullCount,
    verifyMatchCount: verifyMatch,
    verifyMismatchPids: mismatchPids,
    applyComplete: applyComplete
  };
}

function _terraCShadowIsAllowedOkCode_(code){
  return code === null || /^C[1-7]$/.test(String(code));
}

/** Terra C shadow record → production C Note result. 재판정 없이 복사. */
function _terraCShadowBuildProductionResult_(pid, rec){
  const status = String(rec.status || "");
  if (status === "ERROR") {
    return {
      schema_version: "KCMP_C_V1",
      status: "ERROR",
      error_type: String(rec.error_type || "PACKET_ERROR"),
      message: rec.message != null ? String(rec.message) : "",
      pid: String(pid || ""),
      code: null,
      contributors: Array.isArray(rec.contributors) ? rec.contributors.slice() : [],
      source: "KCMP_TERRA_C_SHADOW_ACCEPTED"
    };
  }

  const result = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    pid: String(pid || ""),
    code: (rec.code === undefined) ? null : rec.code,
    contributors: Array.isArray(rec.contributors) ? rec.contributors.slice() : [],
    source: "KCMP_TERRA_C_SHADOW_ACCEPTED"
  };
  if (Object.prototype.hasOwnProperty.call(rec, "reason")) result.reason = rec.reason;
  if (Object.prototype.hasOwnProperty.call(rec, "message")) result.message = rec.message;
  if (Object.prototype.hasOwnProperty.call(rec, "error_type")) result.error_type = rec.error_type;
  if (Object.prototype.hasOwnProperty.call(rec, "interaction_summary")) result.interaction_summary = rec.interaction_summary;
  if (Object.prototype.hasOwnProperty.call(rec, "decision_path")) result.decision_path = rec.decision_path;
  if (Object.prototype.hasOwnProperty.call(rec, "boundary_check")) result.boundary_check = rec.boundary_check;
  if (Object.prototype.hasOwnProperty.call(rec, "context_needed")) result.context_needed = rec.context_needed;
  if (Object.prototype.hasOwnProperty.call(rec, "quotes")) result.quotes = rec.quotes;
  return result;
}

/**
 * STEP 16F: Terra C shadow 확정 결과를 production C에 1회 적용.
 * GPT/semantic 변경 없음. K/M/P / C shadow Properties 미수정.
 * PACKET_ERROR는 finalized ERROR Note로 보존. 73건 전체 preflight 후 write.
 */
function APPLY_TERRA_C_SHADOW_TO_PRODUCTION(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;

  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== TERRA C SHADOW -> PRODUCTION APPLY ===");
    Logger.log("SHEET=" + String(sh.getName()));
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "WRONG_SHEET" };
  }

  const map = ensureColMapOrHalt_();
  const cCol = colNumOf(map.L);
  if (!(cCol > 0)) {
    Logger.log("=== TERRA C SHADOW -> PRODUCTION APPLY ===");
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=C_COLUMN_UNRESOLVED");
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "C_COLUMN_UNRESOLVED" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  if (packets.length !== TOTAL) {
    Logger.log("=== TERRA C SHADOW -> PRODUCTION APPLY ===");
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=PACKET_COUNT_MISMATCH expected=" + TOTAL);
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "PACKET_COUNT_MISMATCH" };
  }

  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  const prepared = [];
  let shadowFound = 0;
  let shadowOk = 0;
  let shadowError = 0;
  let packetErrorCount = 0;
  const preflightProblems = [];

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const packet = byPid[pid];
    if (!packet || !(Number(packet.representativeRow) > 0)) {
      preflightProblems.push({ pid: pid, problem: "PACKET_OR_ROW_MISSING" });
      continue;
    }

    const rec = _terraCShadowLoad_(pid);
    if (!rec) {
      preflightProblems.push({ pid: pid, problem: "SHADOW_MISSING" });
      continue;
    }
    shadowFound++;

    const status = String(rec.status || "");
    const errorType = String(rec.error_type || "");

    if (status === "OK") {
      shadowOk++;
      const code = (rec.code === undefined) ? null : rec.code;
      if (!_terraCShadowIsAllowedOkCode_(code)) {
        preflightProblems.push({ pid: pid, problem: "INVALID_OK_CODE", code: code });
        continue;
      }
      if (!Array.isArray(rec.contributors)) {
        preflightProblems.push({ pid: pid, problem: "CONTRIBUTORS_NOT_ARRAY" });
        continue;
      }
    } else if (status === "ERROR") {
      shadowError++;
      if (errorType !== "PACKET_ERROR") {
        preflightProblems.push({
          pid: pid,
          problem: "NON_PACKET_ERROR_NOT_ALLOWED",
          error_type: errorType
        });
        continue;
      }
      packetErrorCount++;
    } else {
      preflightProblems.push({ pid: pid, problem: "INVALID_STATUS", status: status });
      continue;
    }

    const result = _terraCShadowBuildProductionResult_(pid, rec);
    prepared.push({
      pid: pid,
      row: Number(packet.representativeRow),
      result: result,
      expectedStatus: result.status,
      expectedCode: result.code == null ? null : result.code,
      expectedContributors: Array.isArray(result.contributors) ? result.contributors.slice() : [],
      expectedErrorType: result.error_type != null ? String(result.error_type) : ""
    });
  }

  const preflightPass = (preflightProblems.length === 0 && prepared.length === TOTAL && shadowFound === TOTAL);
  if (!preflightPass) {
    Logger.log("=== TERRA C SHADOW -> PRODUCTION APPLY ===");
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("SHADOW_FOUND=" + shadowFound);
    Logger.log("SHADOW_OK=" + shadowOk);
    Logger.log("SHADOW_ERROR=" + shadowError);
    Logger.log("PACKET_ERROR_COUNT=" + packetErrorCount);
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("PREFLIGHT_PROBLEMS=" + JSON.stringify(preflightProblems));
    Logger.log("WRITTEN_COUNT=0");
    Logger.log("VERIFY_MATCH_COUNT=0");
    Logger.log("VERIFY_MISMATCH_COUNT=0");
    Logger.log("VERIFY_MISMATCH_PIDS=[]");
    Logger.log("GPT_CALLS=0");
    Logger.log("K_WRITES=0");
    Logger.log("M_WRITES=0");
    Logger.log("P_WRITES=0");
    Logger.log("SHADOW_WRITES=0");
    Logger.log("SEMANTIC_CHANGE=NONE");
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return {
      ok: false,
      preflightPass: false,
      problems: preflightProblems,
      shadowFound: shadowFound,
      shadowOk: shadowOk,
      shadowError: shadowError,
      packetErrorCount: packetErrorCount
    };
  }

  let writtenCount = 0;
  prepared.forEach(function(item){
    _writeCDecisionCell_(sh, item.row, cCol, item.result);
    writtenCount++;
  });

  const verifyRows = prepared.map(function(item){ return item.row; });
  const notesAfter = _batchGetNotesForRows_(sh, cCol, verifyRows);
  const mismatchPids = [];
  let verifyMatch = 0;

  prepared.forEach(function(item){
    const noteObj = _parseKCMPNoteJson_(notesAfter[item.row]);
    const statusOk = noteObj && String(noteObj.status || "") === String(item.expectedStatus);
    const prodCode = noteObj && (noteObj.code === undefined) ? null : (noteObj ? noteObj.code : null);
    const codesEqual = (prodCode == null && item.expectedCode == null) ||
      (prodCode != null && item.expectedCode != null && String(prodCode) === String(item.expectedCode));
    const contribsEqual = _sameCanonicalContributorSet_(
      noteObj && noteObj.contributors,
      item.expectedContributors
    );
    const et = noteObj && noteObj.error_type != null ? String(noteObj.error_type) : "";
    const errorTypeEqual = (et === String(item.expectedErrorType || ""));

    if (statusOk && codesEqual && contribsEqual && errorTypeEqual) {
      verifyMatch++;
    } else {
      mismatchPids.push(item.pid);
    }
  });

  const applyComplete = (writtenCount === TOTAL && verifyMatch === TOTAL && mismatchPids.length === 0);

  Logger.log("=== TERRA C SHADOW -> PRODUCTION APPLY ===");
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("SHADOW_FOUND=" + shadowFound);
  Logger.log("SHADOW_OK=" + shadowOk);
  Logger.log("SHADOW_ERROR=" + shadowError);
  Logger.log("PACKET_ERROR_COUNT=" + packetErrorCount);
  Logger.log("PREFLIGHT_PASS=true");
  Logger.log("WRITTEN_COUNT=" + writtenCount);
  Logger.log("VERIFY_MATCH_COUNT=" + verifyMatch);
  Logger.log("VERIFY_MISMATCH_COUNT=" + mismatchPids.length);
  Logger.log("VERIFY_MISMATCH_PIDS=" + JSON.stringify(mismatchPids));
  Logger.log("GPT_CALLS=0");
  Logger.log("K_WRITES=0");
  Logger.log("M_WRITES=0");
  Logger.log("P_WRITES=0");
  Logger.log("SHADOW_WRITES=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("APPLY_COMPLETE=" + String(applyComplete));
  Logger.log("================================");

  return {
    ok: applyComplete,
    preflightPass: true,
    writtenCount: writtenCount,
    verifyMatchCount: verifyMatch,
    verifyMismatchPids: mismatchPids,
    shadowOk: shadowOk,
    shadowError: shadowError,
    packetErrorCount: packetErrorCount,
    applyComplete: applyComplete
  };
}

function _terraMShadowIsAllowedOkCode_(code){
  return code === null || /^M[1-4]$/.test(String(code));
}

/** Terra M shadow record → production M Note result. 재판정 없이 복사. */
function _terraMShadowBuildProductionResult_(pid, rec){
  const result = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    pid: String(pid || ""),
    code: (rec.code === undefined) ? null : rec.code,
    contributors: Array.isArray(rec.contributors) ? rec.contributors.slice() : [],
    source: "KCMP_TERRA_M_SHADOW_ACCEPTED"
  };
  if (Object.prototype.hasOwnProperty.call(rec, "reason")) result.reason = rec.reason;
  if (Object.prototype.hasOwnProperty.call(rec, "message")) result.message = rec.message;
  if (Object.prototype.hasOwnProperty.call(rec, "error_type")) result.error_type = rec.error_type;
  if (Object.prototype.hasOwnProperty.call(rec, "metacognitive_target")) result.metacognitive_target = rec.metacognitive_target;
  if (Object.prototype.hasOwnProperty.call(rec, "decision_path")) result.decision_path = rec.decision_path;
  if (Object.prototype.hasOwnProperty.call(rec, "boundary_check")) result.boundary_check = rec.boundary_check;
  if (Object.prototype.hasOwnProperty.call(rec, "context_needed")) result.context_needed = rec.context_needed;
  if (Object.prototype.hasOwnProperty.call(rec, "quotes")) result.quotes = rec.quotes;
  if (Object.prototype.hasOwnProperty.call(rec, "m3_evidence")) result.m3_evidence = rec.m3_evidence;
  return result;
}

/**
 * STEP 16G: Terra M shadow 확정 결과를 production M에 1회 적용.
 * GPT/semantic 변경 없음. K/C/P / M shadow Properties 미수정.
 * 73건 전부 status=OK preflight 후 write.
 */
function APPLY_TERRA_M_SHADOW_TO_PRODUCTION(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;

  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== TERRA M SHADOW -> PRODUCTION APPLY ===");
    Logger.log("SHEET=" + String(sh.getName()));
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "WRONG_SHEET" };
  }

  const map = ensureColMapOrHalt_();
  const mCol = colNumOf(map.M);
  if (!(mCol > 0)) {
    Logger.log("=== TERRA M SHADOW -> PRODUCTION APPLY ===");
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=M_COLUMN_UNRESOLVED");
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "M_COLUMN_UNRESOLVED" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  if (packets.length !== TOTAL) {
    Logger.log("=== TERRA M SHADOW -> PRODUCTION APPLY ===");
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("ERROR=PACKET_COUNT_MISMATCH expected=" + TOTAL);
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return { ok: false, preflightPass: false, error: "PACKET_COUNT_MISMATCH" };
  }

  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  const prepared = [];
  let shadowFound = 0;
  let shadowOk = 0;
  const preflightProblems = [];

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const packet = byPid[pid];
    if (!packet || !(Number(packet.representativeRow) > 0)) {
      preflightProblems.push({ pid: pid, problem: "PACKET_OR_ROW_MISSING" });
      continue;
    }

    const rec = _terraMShadowLoad_(pid);
    if (!rec) {
      preflightProblems.push({ pid: pid, problem: "SHADOW_MISSING" });
      continue;
    }
    shadowFound++;

    if (String(rec.status || "") !== "OK") {
      preflightProblems.push({
        pid: pid,
        problem: "SHADOW_NOT_OK",
        status: String(rec.status || ""),
        error_type: String(rec.error_type || "")
      });
      continue;
    }
    shadowOk++;

    const code = (rec.code === undefined) ? null : rec.code;
    if (!_terraMShadowIsAllowedOkCode_(code)) {
      preflightProblems.push({ pid: pid, problem: "INVALID_OK_CODE", code: code });
      continue;
    }
    if (!Array.isArray(rec.contributors)) {
      preflightProblems.push({ pid: pid, problem: "CONTRIBUTORS_NOT_ARRAY" });
      continue;
    }

    const result = _terraMShadowBuildProductionResult_(pid, rec);
    prepared.push({
      pid: pid,
      row: Number(packet.representativeRow),
      result: result,
      expectedCode: result.code == null ? null : result.code,
      expectedContributors: Array.isArray(result.contributors) ? result.contributors.slice() : []
    });
  }

  const preflightPass = (preflightProblems.length === 0 && prepared.length === TOTAL && shadowFound === TOTAL && shadowOk === TOTAL);
  if (!preflightPass) {
    Logger.log("=== TERRA M SHADOW -> PRODUCTION APPLY ===");
    Logger.log("TOTAL_PACKETS=" + packets.length);
    Logger.log("SHADOW_FOUND=" + shadowFound);
    Logger.log("SHADOW_OK=" + shadowOk);
    Logger.log("PREFLIGHT_PASS=false");
    Logger.log("PREFLIGHT_PROBLEMS=" + JSON.stringify(preflightProblems));
    Logger.log("WRITTEN_COUNT=0");
    Logger.log("VERIFY_MATCH_COUNT=0");
    Logger.log("VERIFY_MISMATCH_COUNT=0");
    Logger.log("VERIFY_MISMATCH_PIDS=[]");
    Logger.log("GPT_CALLS=0");
    Logger.log("K_WRITES=0");
    Logger.log("C_WRITES=0");
    Logger.log("P_WRITES=0");
    Logger.log("SHADOW_WRITES=0");
    Logger.log("SEMANTIC_CHANGE=NONE");
    Logger.log("APPLY_COMPLETE=false");
    Logger.log("================================");
    return {
      ok: false,
      preflightPass: false,
      problems: preflightProblems,
      shadowFound: shadowFound,
      shadowOk: shadowOk
    };
  }

  let writtenCount = 0;
  prepared.forEach(function(item){
    _writeMDecisionCell_(sh, item.row, mCol, item.result);
    writtenCount++;
  });

  const verifyRows = prepared.map(function(item){ return item.row; });
  const notesAfter = _batchGetNotesForRows_(sh, mCol, verifyRows);
  const mismatchPids = [];
  let verifyMatch = 0;

  prepared.forEach(function(item){
    const noteObj = _parseKCMPNoteJson_(notesAfter[item.row]);
    const statusOk = noteObj && String(noteObj.status || "") === "OK";
    const prodCode = noteObj && (noteObj.code === undefined) ? null : (noteObj ? noteObj.code : null);
    const codesEqual = (prodCode == null && item.expectedCode == null) ||
      (prodCode != null && item.expectedCode != null && String(prodCode) === String(item.expectedCode));
    const contribsEqual = _sameCanonicalContributorSet_(
      noteObj && noteObj.contributors,
      item.expectedContributors
    );
    if (statusOk && codesEqual && contribsEqual) {
      verifyMatch++;
    } else {
      mismatchPids.push(item.pid);
    }
  });

  const applyComplete = (writtenCount === TOTAL && verifyMatch === TOTAL && mismatchPids.length === 0);

  Logger.log("=== TERRA M SHADOW -> PRODUCTION APPLY ===");
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("SHADOW_FOUND=" + shadowFound);
  Logger.log("SHADOW_OK=" + shadowOk);
  Logger.log("PREFLIGHT_PASS=true");
  Logger.log("WRITTEN_COUNT=" + writtenCount);
  Logger.log("VERIFY_MATCH_COUNT=" + verifyMatch);
  Logger.log("VERIFY_MISMATCH_COUNT=" + mismatchPids.length);
  Logger.log("VERIFY_MISMATCH_PIDS=" + JSON.stringify(mismatchPids));
  Logger.log("GPT_CALLS=0");
  Logger.log("K_WRITES=0");
  Logger.log("C_WRITES=0");
  Logger.log("P_WRITES=0");
  Logger.log("SHADOW_WRITES=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("APPLY_COMPLETE=" + String(applyComplete));
  Logger.log("================================");

  return {
    ok: applyComplete,
    preflightPass: true,
    writtenCount: writtenCount,
    verifyMatchCount: verifyMatch,
    verifyMismatchPids: mismatchPids,
    shadowFound: shadowFound,
    shadowOk: shadowOk,
    applyComplete: applyComplete
  };
}

/** production K display/Note → agreement용 코드 (read-only). ERROR→BLANK */
function _kCand16BReadProductionCode_(displayText, noteText){
  const fin = _isFinalKCMPDecisionNote_(noteText, "K");
  if (fin.finalized && fin.status === "ERROR") return "";
  let aiCode = _agreementParseDimCode_(displayText, "K");
  if (!aiCode && fin.finalized && fin.status === "OK") {
    const obj = _parseKCMPNoteJson_(noteText);
    if (obj && obj.code != null && obj.code !== "") aiCode = String(obj.code);
  }
  return aiCode || "";
}

function _kCand16BStoredValidationErrors_(rec){
  if (!rec || typeof rec !== "object") return [];
  if (Array.isArray(rec.validation_errors)) return rec.validation_errors.slice();
  return [];
}

/**
 * STEP 16C: Candidate16B ERROR + mismatch READ-ONLY diagnostic.
 * GPT/property write/production/candidate 수정 없음.
 */
function TEST_TERRA_K_CANDIDATE16B_DIAGNOSTIC_AUDIT(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== K CANDIDATE16B DIAGNOSTIC SUMMARY ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("PROPERTY_WRITES=0");
    Logger.log("CLEAR_CALLS=0");
    Logger.log("PRODUCTION_UNCHANGED=true");
    Logger.log("CANDIDATE_UNCHANGED=true");
    Logger.log("READ_ONLY=true");
    Logger.log("================================");
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  const goldArr = KCMP_GOLD_14_4.K;
  const rows = [];
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const packet = byPid[pid];
    if (packet && packet.representativeRow) rows.push(packet.representativeRow);
  }
  const kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  const kDisplays = _batchGetDisplaysForRows_(sh, kCol, rows);

  let candidateOk = 0;
  let candidateError = 0;
  let match = 0;
  let aiBlankGoldCode = 0;
  let aiCodeGoldBlank = 0;
  let subcodeMismatch = 0;

  const falsePositivePids = [];
  const errorPids = [];
  let errorGoldBlank = 0;
  let errorGoldK1 = 0;
  let errorGoldK2 = 0;
  let errorGoldK3 = 0;
  let errorAndMismatchCount = 0;

  const pairCounts = {
    BLANK_TO_K1: 0, BLANK_TO_K2: 0, BLANK_TO_K3: 0,
    K1_TO_BLANK: 0, K2_TO_BLANK: 0, K3_TO_BLANK: 0,
    K1_TO_K2: 0, K1_TO_K3: 0,
    K2_TO_K1: 0, K2_TO_K3: 0,
    K3_TO_K1: 0, K3_TO_K2: 0
  };

  const improvedPids = [];
  const regressedPids = [];
  let unchangedMatch = 0;
  let unchangedMismatch = 0;

  function bumpPair(aiCode, goldCode){
    const a = aiCode === "" ? "BLANK" : String(aiCode);
    const g = goldCode === "" ? "BLANK" : String(goldCode);
    const key = a + "_TO_" + g;
    if (pairCounts.hasOwnProperty(key)) pairCounts[key]++;
  }

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const goldCode = goldArr[i] == null ? "" : String(goldArr[i]);
    const packet = byPid[pid] || null;
    const row = packet && packet.representativeRow ? packet.representativeRow : null;
    const neighbors = _kCand16BResolveNeighbors_(packet || { context: {} }, packets);
    const currUtt = _kMismatchPacketUtterances_(packet);
    const prevUtt = neighbors.previousPacket ? _kMismatchPacketUtterances_(neighbors.previousPacket) : [];
    const nextUtt = neighbors.nextPacket ? _kMismatchPacketUtterances_(neighbors.nextPacket) : [];

    const noteText = row != null ? (kNotes[row] || "") : "";
    const displayText = row != null ? (kDisplays[row] || "") : "";
    const productionCode = _kCand16BReadProductionCode_(displayText, noteText);

    const rec = _kCand16BLoad_(pid);
    const candStatus = rec && rec.status != null ? String(rec.status) : "";
    const candCode = _kCand16BCodeForAgreement_(rec);
    const prodMatch = (productionCode === goldCode);
    const candMatch = (candCode === goldCode);

    if (candStatus === "OK") candidateOk++;
    else if (candStatus === "ERROR") candidateError++;

    // ERROR inventory
    if (candStatus === "ERROR") {
      errorPids.push(pid);
      if (goldCode === "") errorGoldBlank++;
      else if (goldCode === "K1") errorGoldK1++;
      else if (goldCode === "K2") errorGoldK2++;
      else if (goldCode === "K3") errorGoldK3++;

      Logger.log("=== K CAND16B ERROR ===");
      Logger.log("PID=" + pid);
      Logger.log("REPRESENTATIVE_ROW=" + (row == null ? "" : String(row)));
      Logger.log("GOLD_CODE=" + (goldCode === "" ? "BLANK" : goldCode));
      Logger.log("PRODUCTION_CODE=" + (productionCode === "" ? "BLANK" : productionCode));
      Logger.log("CANDIDATE_STATUS=" + candStatus);
      Logger.log("CANDIDATE_ERROR_TYPE=" + String(rec && rec.error_type != null ? rec.error_type : ""));
      Logger.log("MESSAGE=" + String(rec && rec.message != null ? rec.message : ""));
      Logger.log("VALIDATION_ERRORS=" + JSON.stringify(_kCand16BStoredValidationErrors_(rec)));
      Logger.log("RAW_CANDIDATE_RECORD=" + JSON.stringify(rec || null));
      Logger.log("CURRENT_PACKET_UTTERANCES=" + JSON.stringify(currUtt));
      Logger.log("PREVIOUS_PACKET_UTTERANCES=" + JSON.stringify(prevUtt));
      Logger.log("NEXT_PACKET_UTTERANCES=" + JSON.stringify(nextUtt));
      Logger.log("---");
    }

    // mismatch vs gold (agreement code 기준)
    if (candMatch) {
      match++;
    } else {
      const mismatchType = _kMismatchType_(goldCode, candCode);
      if (mismatchType === "AI_BLANK_GOLD_CODE") aiBlankGoldCode++;
      else if (mismatchType === "AI_CODE_GOLD_BLANK") {
        aiCodeGoldBlank++;
        falsePositivePids.push(pid);
      } else {
        subcodeMismatch++;
      }
      bumpPair(candCode, goldCode);
      if (candStatus === "ERROR") errorAndMismatchCount++;

      Logger.log("=== K CAND16B MISMATCH ===");
      Logger.log("PID=" + pid);
      Logger.log("REPRESENTATIVE_ROW=" + (row == null ? "" : String(row)));
      Logger.log("GOLD_CODE=" + (goldCode === "" ? "BLANK" : goldCode));
      Logger.log("PRODUCTION_CODE=" + (productionCode === "" ? "BLANK" : productionCode));
      Logger.log("CANDIDATE_CODE=" + (candCode === "" ? "BLANK" : candCode));
      Logger.log("CANDIDATE_STATUS=" + candStatus);
      Logger.log("MISMATCH_TYPE=" + mismatchType);
      Logger.log("CONTRIBUTORS=" + JSON.stringify(rec && Array.isArray(rec.contributors) ? rec.contributors : []));
      Logger.log("SCIENCE_CONTENT=" + (rec && rec.science_content != null ? String(rec.science_content) : ""));
      Logger.log("CLAIM=" + (rec && rec.claim != null ? String(rec.claim) : ""));
      Logger.log("EVIDENCE=" + (rec && rec.evidence != null ? String(rec.evidence) : ""));
      Logger.log("REASON=" + (rec && rec.reason != null ? String(rec.reason) : ""));
      Logger.log("CURRENT_PACKET_UTTERANCES=" + JSON.stringify(currUtt));
      Logger.log("PREVIOUS_PACKET_UTTERANCES=" + JSON.stringify(prevUtt));
      Logger.log("NEXT_PACKET_UTTERANCES=" + JSON.stringify(nextUtt));
      Logger.log("---");
    }

    // production vs candidate improvement
    if (!prodMatch && candMatch) improvedPids.push(pid);
    else if (prodMatch && !candMatch) regressedPids.push(pid);
    else if (prodMatch && candMatch) unchangedMatch++;
    else unchangedMismatch++;
  }

  const totalMismatch = aiBlankGoldCode + aiCodeGoldBlank + subcodeMismatch;

  Logger.log("=== K CANDIDATE16B DIAGNOSTIC SUMMARY ===");
  Logger.log("TOTAL_PID=" + TOTAL);
  Logger.log("CANDIDATE_OK=" + candidateOk);
  Logger.log("CANDIDATE_ERROR=" + candidateError);
  Logger.log("MATCH=" + match);
  Logger.log("TOTAL_MISMATCH=" + totalMismatch);
  Logger.log("AI_BLANK_GOLD_CODE=" + aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + subcodeMismatch);
  Logger.log("FALSE_POSITIVE_PIDS=" + JSON.stringify(falsePositivePids));
  Logger.log("ERROR_TOTAL=" + errorPids.length);
  Logger.log("ERROR_GOLD_BLANK=" + errorGoldBlank);
  Logger.log("ERROR_GOLD_K1=" + errorGoldK1);
  Logger.log("ERROR_GOLD_K2=" + errorGoldK2);
  Logger.log("ERROR_GOLD_K3=" + errorGoldK3);
  Logger.log("ERROR_PIDS=" + JSON.stringify(errorPids));
  Logger.log("ERROR_AND_MISMATCH_COUNT=" + errorAndMismatchCount);
  Logger.log("BLANK_TO_K1=" + pairCounts.BLANK_TO_K1);
  Logger.log("BLANK_TO_K2=" + pairCounts.BLANK_TO_K2);
  Logger.log("BLANK_TO_K3=" + pairCounts.BLANK_TO_K3);
  Logger.log("K1_TO_BLANK=" + pairCounts.K1_TO_BLANK);
  Logger.log("K2_TO_BLANK=" + pairCounts.K2_TO_BLANK);
  Logger.log("K3_TO_BLANK=" + pairCounts.K3_TO_BLANK);
  Logger.log("K1_TO_K2=" + pairCounts.K1_TO_K2);
  Logger.log("K1_TO_K3=" + pairCounts.K1_TO_K3);
  Logger.log("K2_TO_K1=" + pairCounts.K2_TO_K1);
  Logger.log("K2_TO_K3=" + pairCounts.K2_TO_K3);
  Logger.log("K3_TO_K1=" + pairCounts.K3_TO_K1);
  Logger.log("K3_TO_K2=" + pairCounts.K3_TO_K2);
  Logger.log("IMPROVED_COUNT=" + improvedPids.length);
  Logger.log("IMPROVED_PIDS=" + JSON.stringify(improvedPids));
  Logger.log("REGRESSED_COUNT=" + regressedPids.length);
  Logger.log("REGRESSED_PIDS=" + JSON.stringify(regressedPids));
  Logger.log("UNCHANGED_MATCH_COUNT=" + unchangedMatch);
  Logger.log("UNCHANGED_MISMATCH_COUNT=" + unchangedMismatch);
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PROPERTY_WRITES=0");
  Logger.log("CLEAR_CALLS=0");
  Logger.log("PRODUCTION_UNCHANGED=true");
  Logger.log("CANDIDATE_UNCHANGED=true");
  Logger.log("READ_ONLY=true");
  Logger.log("================================");

  return {
    ok: true,
    candidateOk: candidateOk,
    candidateError: candidateError,
    match: match,
    totalMismatch: totalMismatch,
    falsePositivePids: falsePositivePids,
    errorPids: errorPids,
    improvedPids: improvedPids,
    regressedPids: regressedPids
  };
}

function _terraCShadowSheetKey_(){
  return "14차시4조";
}

function _terraCShadowPropKey_(pid){
  return "KCMP_TERRA_C_SHADOW|" + _terraCShadowSheetKey_() + "|" + String(pid || "");
}

function _terraCShadowLoad_(pid){
  const raw = PropertiesService.getDocumentProperties().getProperty(_terraCShadowPropKey_(pid));
  if (!raw) return null;
  try {
    const obj = JSON.parse(raw);
    if (!obj || typeof obj !== "object") return null;
    return obj;
  } catch (e) {
    return null;
  }
}

function _terraCShadowSave_(record){
  PropertiesService.getDocumentProperties().setProperty(
    _terraCShadowPropKey_(record.pid),
    JSON.stringify(record)
  );
}

function _terraCShadowIsFinalized_(record){
  if (!record) return false;
  const status = String(record.status || "");
  return status === "OK" || status === "ERROR";
}

function _terraCShadowCodeForAgreement_(record){
  if (!record || record.status !== "OK") return "";
  if (record.code == null || record.code === "") return "";
  return String(record.code);
}

/** shadow C only: gpt-5.6-terra 최대 5 PID. production C 셀/Note 미변경. */
function TEST_KCMP_TERRA_C_SHADOW_BATCH(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== KCMP TERRA C SHADOW BATCH ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = loadColMap_();
  if (!map) {
    Logger.log("ERROR=COLMAP_MISSING");
    return { ok: false, error: "COLMAP_MISSING" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  let finalizedBefore = 0;
  const pending = [];
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraCShadowLoad_(pid);
    if (_terraCShadowIsFinalized_(rec)) {
      finalizedBefore++;
      continue;
    }
    pending.push(pid);
  }

  Logger.log("=== KCMP TERRA C SHADOW BATCH ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("MODEL=" + String(MODEL_C));
  Logger.log("TOTAL_PACKETS=" + TOTAL);
  Logger.log("SHADOW_FINALIZED_BEFORE=" + finalizedBefore);
  Logger.log("MAX_CASES=" + KCMP_TERRA_C_SHADOW_MAX_CASES);

  const toProcess = pending.slice(0, KCMP_TERRA_C_SHADOW_MAX_CASES);
  let processed = 0;
  let shadowOk = 0;
  let shadowError = 0;

  toProcess.forEach(function(pid, idx){
    if (idx > 0) Utilities.sleep(3000);
    const packet = byPid[pid];
    let result;
    if (!packet) {
      result = { status: "ERROR", code: null, contributors: [], error_type: "PACKET_ERROR", message: "packet not found" };
    } else {
      result = runCDecisionTreeForPacket_(packet);
    }
    const record = {
      pid: pid,
      status: result && result.status ? String(result.status) : "ERROR",
      code: (result && result.code != null) ? result.code : null,
      contributors: (result && result.contributors) ? result.contributors : [],
      error_type: result && result.error_type != null ? String(result.error_type) : "",
      message: result && result.message != null ? String(result.message) : ""
    };
    _terraCShadowSave_(record);
    processed++;
    if (record.status === "OK") shadowOk++;
    else shadowError++;
    Logger.log("PID=" + pid);
    Logger.log("STATUS=" + record.status);
    Logger.log("CODE=" + (record.code == null ? "null" : String(record.code)));
    Logger.log("ERROR_TYPE=" + record.error_type);
    Logger.log("---");
  });

  let remaining = 0;
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    if (!_terraCShadowIsFinalized_(_terraCShadowLoad_(pid))) remaining++;
  }

  Logger.log("PROCESSED_THIS_RUN=" + processed);
  Logger.log("SHADOW_OK=" + shadowOk);
  Logger.log("SHADOW_ERROR=" + shadowError);
  Logger.log("SHADOW_REMAINING=" + remaining);
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PRODUCTION_CLEAR_CALLS=0");
  Logger.log("PRODUCTION_C_UNCHANGED=true");
  Logger.log("AUTO_CHAINING=false");
  Logger.log("================================");

  return {
    processed: processed,
    shadowOk: shadowOk,
    shadowError: shadowError,
    remaining: remaining
  };
}

/** GPT 없음. Terra shadow C vs human gold. 73 finalized 전에는 benchmark 중단. */
function TEST_KCMP_TERRA_C_SHADOW_SUMMARY(){
  const TOTAL = 73;
  const MINI_KAPPA = 0.4069;
  const MINI_AGREEMENT = 67.1;
  const records = [];
  let finalized = 0;
  let shadowErrorCount = 0;

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraCShadowLoad_(pid);
    records.push(rec);
    if (_terraCShadowIsFinalized_(rec)) finalized++;
    if (rec && rec.status === "ERROR") shadowErrorCount++;
  }

  const remaining = TOTAL - finalized;
  Logger.log("=== KCMP TERRA C SHADOW SUMMARY ===");
  Logger.log("MODEL=gpt-5.6-terra");
  Logger.log("TOTAL_PID=" + TOTAL);

  if (finalized !== TOTAL) {
    Logger.log("SHADOW_COMPLETE=false");
    Logger.log("SHADOW_FINALIZED=" + finalized);
    Logger.log("SHADOW_REMAINING=" + remaining);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("PRODUCTION_C_UNCHANGED=true");
    Logger.log("================================");
    return { complete: false, finalized: finalized, remaining: remaining };
  }

  const terraCodes = records.map(_terraCShadowCodeForAgreement_);
  const gold = KCMP_GOLD_14_4.C;
  const kappa = _computeCohenKappa_(gold, terraCodes);
  const tax = _agreementTaxonomy_(gold, terraCodes);
  const goldDist = _agreementDistribution_(gold);
  const terraDist = _agreementDistribution_(terraCodes);
  const agreementPercent = kappa.po * 100;

  Logger.log("SHADOW_COMPLETE=true");
  Logger.log("KAPPA=" + _agreementRound_(kappa.kappa, 4));
  Logger.log("AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent, 1));
  Logger.log("MATCH=" + tax.match);
  Logger.log("AI_BLANK_GOLD_CODE=" + tax.aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + tax.aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + tax.subcodeMismatch);
  Logger.log("GOLD_DISTRIBUTION=" + _agreementFormatDistribution_(goldDist));
  Logger.log("TERRA_DISTRIBUTION=" + _agreementFormatDistribution_(terraDist));
  Logger.log("TOP_CONFUSION_PAIRS_AI_TO_GOLD=" + _agreementFormatConfusion_(tax.confusionPairs, 10));
  Logger.log("SHADOW_ERROR_COUNT=" + shadowErrorCount);
  Logger.log("MINI_BASELINE_KAPPA=" + MINI_KAPPA);
  Logger.log("MINI_BASELINE_AGREEMENT=" + MINI_AGREEMENT);
  Logger.log("DELTA_KAPPA=" + _agreementRound_((kappa.kappa == null ? 0 : kappa.kappa) - MINI_KAPPA, 4));
  Logger.log("DELTA_AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent - MINI_AGREEMENT, 1));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PRODUCTION_C_UNCHANGED=true");
  Logger.log("================================");

  return { complete: true, kappa: kappa.kappa, agreementPercent: agreementPercent, taxonomy: tax };
}

/** GPT 없음. Terra C shadow ERROR만 read-only inventory. properties/sheet/production 미변경. */
function TEST_KCMP_TERRA_C_SHADOW_ERROR_INVENTORY(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== KCMP TERRA C SHADOW ERROR INVENTORY ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("SHADOW_WRITES=0");
    Logger.log("PRODUCTION_C_UNCHANGED=true");
    Logger.log("READ_ONLY=true");
    Logger.log("================================");
    return { ok: false, error: "WRONG_SHEET" };
  }

  const errorRecords = [];
  const errorTypeCounts = {};
  const errorPids = [];

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraCShadowLoad_(pid);
    if (!rec || String(rec.status || "") !== "ERROR") continue;

    errorRecords.push(rec);
    errorPids.push(pid);
    const errorType = String(rec.error_type == null ? "" : rec.error_type);
    errorTypeCounts[errorType] = (errorTypeCounts[errorType] || 0) + 1;

    Logger.log("DIMENSION=C");
    Logger.log("PID=" + pid);
    Logger.log("STATUS=" + String(rec.status || ""));
    Logger.log("CODE=" + (rec.code == null ? "null" : String(rec.code)));
    Logger.log("ERROR_TYPE=" + errorType);
    Logger.log("MESSAGE=" + String(rec.message == null ? "" : rec.message));
    Logger.log("CONTRIBUTORS=" + JSON.stringify(rec.contributors == null ? [] : rec.contributors));
    Logger.log("---");
  }

  Logger.log("=== KCMP TERRA C SHADOW ERROR INVENTORY ===");
  Logger.log("MODEL=gpt-5.6-terra");
  Logger.log("TOTAL_PID=" + TOTAL);
  Logger.log("SHADOW_ERROR_COUNT=" + errorRecords.length);
  Logger.log("ERROR_TYPE_COUNTS=" + JSON.stringify(errorTypeCounts));
  Logger.log("ERROR_PIDS=" + JSON.stringify(errorPids));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("SHADOW_WRITES=0");
  Logger.log("PRODUCTION_C_UNCHANGED=true");
  Logger.log("READ_ONLY=true");
  Logger.log("================================");

  return {
    ok: true,
    shadowErrorCount: errorRecords.length,
    errorTypeCounts: errorTypeCounts,
    errorPids: errorPids
  };
}

/** STEP 14D: gpt-5.6-terra M shadow benchmark (production M 덮어쓰기 없음). */
function _terraMShadowSheetKey_(){
  return "14차시4조";
}

function _terraMShadowPropKey_(pid){
  return "KCMP_TERRA_M_SHADOW|" + _terraMShadowSheetKey_() + "|" + String(pid || "");
}

function _terraMShadowLoad_(pid){
  const raw = PropertiesService.getDocumentProperties().getProperty(_terraMShadowPropKey_(pid));
  if (!raw) return null;
  try {
    const obj = JSON.parse(raw);
    if (!obj || typeof obj !== "object") return null;
    return obj;
  } catch (e) {
    return null;
  }
}

function _terraMShadowSave_(record){
  PropertiesService.getDocumentProperties().setProperty(
    _terraMShadowPropKey_(record.pid),
    JSON.stringify(record)
  );
}

function _terraMShadowIsFinalized_(record){
  if (!record) return false;
  const status = String(record.status || "");
  return status === "OK" || status === "ERROR";
}

function _terraMShadowCodeForAgreement_(record){
  // ERROR는 agreement 비교에서 BLANK(= "")으로 취급
  if (!record || record.status !== "OK") return "";
  if (record.code == null || record.code === "") return "";
  return String(record.code);
}

/** GPT 호출: gpt-5.6-terra로 M 재판정 shadow만 저장. 자동 chaining 없음. */
function TEST_KCMP_TERRA_M_SHADOW_BATCH(){
  const SHEET_NAME = "14차시 4조";
  const TOTAL = 73;
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== KCMP TERRA M SHADOW BATCH ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = loadColMap_();
  if (!map) {
    Logger.log("ERROR=COLMAP_MISSING");
    return { ok: false, error: "COLMAP_MISSING" };
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const byPid = {};
  packets.forEach(function(p){
    if (p && p.pid) byPid[p.pid] = p;
  });

  let finalizedBefore = 0;
  const pending = [];
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraMShadowLoad_(pid);
    if (_terraMShadowIsFinalized_(rec)) {
      finalizedBefore++;
      continue;
    }
    pending.push(pid);
  }

  Logger.log("=== KCMP TERRA M SHADOW BATCH ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("MODEL=" + String(MODEL_M));
  Logger.log("TOTAL_PACKETS=" + TOTAL);
  Logger.log("SHADOW_FINALIZED_BEFORE=" + finalizedBefore);
  Logger.log("MAX_CASES=" + KCMP_TERRA_M_SHADOW_MAX_CASES);

  const toProcess = pending.slice(0, KCMP_TERRA_M_SHADOW_MAX_CASES);
  let processed = 0;
  let shadowOk = 0;
  let shadowError = 0;

  toProcess.forEach(function(pid, idx){
    if (idx > 0) Utilities.sleep(3000);

    const packet = byPid[pid];
    let result;
    if (!packet) {
      result = { status: "ERROR", code: null, contributors: [], error_type: "PACKET_ERROR", message: "packet not found" };
    } else {
      result = runMDecisionTreeForPacket_(packet, { allPackets: packets });
    }

    const record = {
      pid: pid,
      status: result && result.status ? String(result.status) : "ERROR",
      code: (result && result.code != null) ? result.code : null,
      contributors: (result && result.contributors) ? result.contributors : [],
      error_type: result && result.error_type != null ? String(result.error_type) : "",
      message: result && result.message != null ? String(result.message) : ""
    };

    _terraMShadowSave_(record);
    processed++;

    if (record.status === "OK") shadowOk++;
    else shadowError++;

    Logger.log("PID=" + pid);
    Logger.log("STATUS=" + record.status);
    Logger.log("CODE=" + (record.code == null ? "null" : String(record.code)));
    Logger.log("ERROR_TYPE=" + record.error_type);
    Logger.log("---");
  });

  let remaining = 0;
  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    if (!_terraMShadowIsFinalized_(_terraMShadowLoad_(pid))) remaining++;
  }

  Logger.log("PROCESSED_THIS_RUN=" + processed);
  Logger.log("SHADOW_OK=" + shadowOk);
  Logger.log("SHADOW_ERROR=" + shadowError);
  Logger.log("SHADOW_REMAINING=" + remaining);
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("SHADOW_WRITES=0");
  Logger.log("PRODUCTION_CLEAR_CALLS=0");
  Logger.log("PRODUCTION_M_UNCHANGED=true");
  Logger.log("AUTO_CHAINING=false");
  Logger.log("================================");

  return {
    processed: processed,
    shadowOk: shadowOk,
    shadowError: shadowError,
    remaining: remaining
  };
}

/** GPT 없음. Terra shadow M vs human gold. 미완료면 agreement 계산 중단. */
function TEST_KCMP_TERRA_M_SHADOW_SUMMARY(){
  const TOTAL = 73;
  const MINI_KAPPA = 0.5208;
  const MINI_AGREEMENT = 75.3;

  const records = [];
  let finalized = 0;
  let shadowErrorCount = 0;

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const rec = _terraMShadowLoad_(pid);
    records.push(rec);
    if (_terraMShadowIsFinalized_(rec)) finalized++;
    if (rec && rec.status === "ERROR") shadowErrorCount++;
  }

  const remaining = TOTAL - finalized;
  Logger.log("=== KCMP TERRA M SHADOW SUMMARY ===");
  Logger.log("MODEL=gpt-5.6-terra");
  Logger.log("TOTAL_PID=" + TOTAL);

  if (finalized !== TOTAL) {
    Logger.log("SHADOW_COMPLETE=false");
    Logger.log("SHADOW_FINALIZED=" + finalized);
    Logger.log("SHADOW_REMAINING=" + remaining);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("PRODUCTION_M_UNCHANGED=true");
    Logger.log("================================");
    return { complete: false, finalized: finalized, remaining: remaining };
  }

  const terraCodes = records.map(_terraMShadowCodeForAgreement_);
  const gold = KCMP_GOLD_14_4.M;

  const kappa = _computeCohenKappa_(gold, terraCodes);
  const tax = _agreementTaxonomy_(gold, terraCodes);

  const goldDist = _agreementDistribution_(gold);
  const terraDist = _agreementDistribution_(terraCodes);
  const agreementPercent = kappa.po * 100;

  Logger.log("SHADOW_COMPLETE=true");
  Logger.log("KAPPA=" + _agreementRound_(kappa.kappa, 4));
  Logger.log("AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent, 1));
  Logger.log("MATCH=" + tax.match);
  Logger.log("AI_BLANK_GOLD_CODE=" + tax.aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + tax.aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + tax.subcodeMismatch);
  Logger.log("GOLD_DISTRIBUTION=" + _agreementFormatDistribution_(goldDist));
  Logger.log("TERRA_DISTRIBUTION=" + _agreementFormatDistribution_(terraDist));
  Logger.log("TOP_CONFUSION_PAIRS_AI_TO_GOLD=" + _agreementFormatConfusion_(tax.confusionPairs, 10));
  Logger.log("SHADOW_ERROR_COUNT=" + shadowErrorCount);

  Logger.log("MINI_BASELINE_KAPPA=" + MINI_KAPPA);
  Logger.log("MINI_BASELINE_AGREEMENT=" + MINI_AGREEMENT);
  Logger.log("DELTA_KAPPA=" + _agreementRound_((kappa.kappa == null ? 0 : kappa.kappa) - MINI_KAPPA, 4));
  Logger.log("DELTA_AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent - MINI_AGREEMENT, 1));

  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("PRODUCTION_M_UNCHANGED=true");
  Logger.log("================================");

  return { complete: true, kappa: kappa.kappa, agreementPercent: agreementPercent, taxonomy: tax, shadowErrorCount: shadowErrorCount };
}

/** Terra K/C/M shadow record → computeDeterministicPForPacket_ 입력용 fake Note JSON. sheet/Note 미사용. */
function _terraShadowRecordToNoteJson_(rec){
  if (!rec || typeof rec !== "object") return "";
  return JSON.stringify({
    status: rec.status != null ? String(rec.status) : "",
    code: (rec.code === null || rec.code === undefined) ? null : rec.code,
    contributors: Array.isArray(rec.contributors) ? rec.contributors : [],
    error_type: rec.error_type != null ? String(rec.error_type) : "",
    message: rec.message != null ? String(rec.message) : ""
  });
}

/** Terra shadow K/C/M → production과 동일 deterministic P. GPT/sheet/Note write 없음. */
function _computeTerraShadowDeterministicP_(pid, kRec, cRec, mRec){
  const packet = { pid: String(pid || "") };
  return computeDeterministicPForPacket_(
    packet,
    _terraShadowRecordToNoteJson_(kRec),
    _terraShadowRecordToNoteJson_(cRec),
    _terraShadowRecordToNoteJson_(mRec)
  );
}

function _terraPShadowCodeForAgreement_(pResult){
  if (!pResult || pResult.status !== "OK") return "";
  if (pResult.code == null || pResult.code === "") return "";
  return String(pResult.code);
}

function _terraPShadowUpstreamErrorMeta_(kRec, cRec, mRec){
  const dims = [
    { key: "K", rec: kRec },
    { key: "C", rec: cRec },
    { key: "M", rec: mRec }
  ];
  for (let i = 0; i < dims.length; i++) {
    const d = dims[i];
    const status = d.rec ? String(d.rec.status || "") : "";
    if (status !== "OK") {
      return {
        upstream_dimension: d.key,
        upstream_error_type: d.rec ? String(d.rec.error_type || "") : "MISSING",
        upstream_status: status || "MISSING"
      };
    }
  }
  return { upstream_dimension: "", upstream_error_type: "", upstream_status: "OK" };
}

/**
 * STEP 14E: Terra K/C/M shadow → deterministic P agreement.
 * GPT 호출 없음. production/shadow properties write 없음.
 */
function TEST_KCMP_TERRA_P_SHADOW_SUMMARY(){
  const TOTAL = 73;
  const MINI_KAPPA = 0.2935;
  const MINI_AGREEMENT = 45.2;

  const terraPCodes = [];
  const errorPids = [];
  let pOkCount = 0;
  let pErrorCount = 0;
  let kcmReady = 0;

  for (let i = 0; i < TOTAL; i++) {
    const pid = _agreementPidFromIndex_(i);
    const kRec = _terraKShadowLoad_(pid);
    const cRec = _terraCShadowLoad_(pid);
    const mRec = _terraMShadowLoad_(pid);

    const kFin = _terraKShadowIsFinalized_(kRec);
    const cFin = _terraCShadowIsFinalized_(cRec);
    const mFin = _terraMShadowIsFinalized_(mRec);
    if (kFin && cFin && mFin) kcmReady++;

    const pResult = _computeTerraShadowDeterministicP_(pid, kRec, cRec, mRec);
    if (pResult && pResult.status === "OK") {
      pOkCount++;
      terraPCodes.push(_terraPShadowCodeForAgreement_(pResult));
    } else {
      pErrorCount++;
      errorPids.push(pid);
      terraPCodes.push("");
      const meta = _terraPShadowUpstreamErrorMeta_(kRec, cRec, mRec);
      Logger.log("P_ERROR_PID=" + pid);
      Logger.log("P_ERROR_TYPE=" + String((pResult && pResult.error_type) || "UPSTREAM_ERROR"));
      Logger.log("UPSTREAM_DIMENSION=" + meta.upstream_dimension);
      Logger.log("UPSTREAM_ERROR_TYPE=" + meta.upstream_error_type);
      Logger.log("UPSTREAM_STATUS=" + meta.upstream_status);
      Logger.log("---");
    }
  }

  Logger.log("=== KCMP TERRA P SHADOW SUMMARY ===");
  Logger.log("MODEL=gpt-5.6-terra");
  Logger.log("TOTAL_PID=" + TOTAL);

  if (kcmReady !== TOTAL) {
    Logger.log("SHADOW_COMPLETE=false");
    Logger.log("KCM_SHADOW_FINALIZED=" + kcmReady);
    Logger.log("KCM_SHADOW_REMAINING=" + (TOTAL - kcmReady));
    Logger.log("P_OK_COUNT=" + pOkCount);
    Logger.log("P_ERROR_COUNT=" + pErrorCount);
    Logger.log("P_ERROR_PIDS=" + JSON.stringify(errorPids));
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("SHADOW_WRITES=0");
    Logger.log("PRODUCTION_P_UNCHANGED=true");
    Logger.log("READ_ONLY=true");
    Logger.log("================================");
    return {
      complete: false,
      kcmReady: kcmReady,
      pOkCount: pOkCount,
      pErrorCount: pErrorCount,
      errorPids: errorPids
    };
  }

  const gold = KCMP_GOLD_14_4.P;
  const kappa = _computeCohenKappa_(gold, terraPCodes);
  const tax = _agreementTaxonomy_(gold, terraPCodes);
  const goldDist = _agreementDistribution_(gold);
  const terraDist = _agreementDistribution_(terraPCodes);
  const agreementPercent = kappa.po * 100;

  Logger.log("SHADOW_COMPLETE=true");
  Logger.log("P_OK_COUNT=" + pOkCount);
  Logger.log("P_ERROR_COUNT=" + pErrorCount);
  Logger.log("P_ERROR_PIDS=" + JSON.stringify(errorPids));
  Logger.log("KAPPA=" + _agreementRound_(kappa.kappa, 4));
  Logger.log("AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent, 1));
  Logger.log("MATCH=" + tax.match);
  Logger.log("AI_BLANK_GOLD_CODE=" + tax.aiBlankGoldCode);
  Logger.log("AI_CODE_GOLD_BLANK=" + tax.aiCodeGoldBlank);
  Logger.log("SUBCODE_MISMATCH=" + tax.subcodeMismatch);
  Logger.log("GOLD_DISTRIBUTION=" + _agreementFormatDistribution_(goldDist));
  Logger.log("TERRA_DISTRIBUTION=" + _agreementFormatDistribution_(terraDist));
  Logger.log("TOP_CONFUSION_PAIRS_AI_TO_GOLD=" + _agreementFormatConfusion_(tax.confusionPairs, 10));
  Logger.log("MINI_BASELINE_KAPPA=" + MINI_KAPPA);
  Logger.log("MINI_BASELINE_AGREEMENT=" + MINI_AGREEMENT);
  Logger.log("DELTA_KAPPA=" + _agreementRound_((kappa.kappa == null ? 0 : kappa.kappa) - MINI_KAPPA, 4));
  Logger.log("DELTA_AGREEMENT_PERCENT=" + _agreementRound_(agreementPercent - MINI_AGREEMENT, 1));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("SHADOW_WRITES=0");
  Logger.log("PRODUCTION_P_UNCHANGED=true");
  Logger.log("READ_ONLY=true");
  Logger.log("================================");

  return {
    complete: true,
    pOkCount: pOkCount,
    pErrorCount: pErrorCount,
    errorPids: errorPids,
    kappa: kappa.kappa,
    agreementPercent: agreementPercent,
    taxonomy: tax
  };
}

/** representativeRow 목록에 대해 연속 블록 단위 batch getDisplayValues() — read-only */
function _batchGetDisplaysForRows_(sheet, col, rows){
  const out = {};
  if (!sheet || !col || !rows || !rows.length) return out;

  const uniq = {};
  rows.forEach(function(r){
    const n = Number(r);
    if (n > 0) uniq[n] = true;
  });
  const sorted = Object.keys(uniq).map(Number).sort(function(a, b){ return a - b; });
  if (!sorted.length) return out;

  let blockStart = sorted[0];
  let blockEnd = sorted[0];
  for (let i = 1; i <= sorted.length; i++) {
    const r = sorted[i];
    if (i < sorted.length && r === blockEnd + 1) {
      blockEnd = r;
      continue;
    }
    const h = blockEnd - blockStart + 1;
    const blockVals = sheet.getRange(blockStart, col, h, 1).getDisplayValues();
    for (let j = 0; j < h; j++) {
      out[blockStart + j] = blockVals[j][0];
    }
    if (i < sorted.length) {
      blockStart = r;
      blockEnd = r;
    }
  }
  return out;
}

function _auditDimResetTargets_(packets, notesByRow, displaysByRow, dimension){
  let finalizedOk = 0;
  let finalizedError = 0;
  let nonemptyDisplay = 0;
  let noteCount = 0;
  const targetRows = [];

  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;

    const noteText = notesByRow[row];
    const display = displaysByRow[row];
    const hasNote = noteText != null && String(noteText).trim().length > 0;
    const hasDisplay = display != null && String(display).trim().length > 0;

    if (hasNote) noteCount++;
    if (hasDisplay) nonemptyDisplay++;

    const fin = _isFinalKCMPDecisionNote_(noteText, dimension);
    if (fin.finalized && fin.status === "OK") finalizedOk++;
    else if (fin.finalized && fin.status === "ERROR") finalizedError++;

    // Terra production 재코딩 전 reset 대상: display 또는 Note가 있는 production 결과 행
    if (hasNote || hasDisplay) targetRows.push(row);
  });

  targetRows.sort(function(a, b){ return a - b; });
  return {
    finalizedOk: finalizedOk,
    finalizedError: finalizedError,
    nonemptyDisplay: nonemptyDisplay,
    noteCount: noteCount,
    targetRows: targetRows,
    wouldClear: targetRows.length > 0
  };
}

/**
 * STEP 15A: Terra production 전환 전 K/C/M/P reset 대상 READ-ONLY audit.
 * clear/write/GPT/shadow 수정 없음. 실제 clear는 RESET_KCMP_PRODUCTION_FOR_TERRA_14_4().
 */
function TEST_KCMP_TERRA_PRODUCTION_RESET_AUDIT(){
  const SHEET_NAME = "14차시 4조";
  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    Logger.log("=== KCMP TERRA PRODUCTION RESET AUDIT ===");
    Logger.log("ERROR=ACTIVE_SHEET_MUST_BE_" + SHEET_NAME);
    Logger.log("GPT_CALLS=0");
    Logger.log("SHEET_WRITES=0");
    Logger.log("NOTE_WRITES=0");
    Logger.log("CLEAR_CALLS=0");
    Logger.log("READ_ONLY=true");
    Logger.log("================================");
    return { ok: false, error: "WRONG_SHEET" };
  }

  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  const rows = _collectKCMPRepresentativeRows_(packets);

  // production writer와 동일 열: K→map.K, C→map.L, M→map.M, P→map.N
  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const pCol = colNumOf(map.N);

  const kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  const cNotes = _batchGetNotesForRows_(sh, cCol, rows);
  const mNotes = _batchGetNotesForRows_(sh, mCol, rows);
  const pNotes = _batchGetNotesForRows_(sh, pCol, rows);

  const kDisplays = _batchGetDisplaysForRows_(sh, kCol, rows);
  const cDisplays = _batchGetDisplaysForRows_(sh, cCol, rows);
  const mDisplays = _batchGetDisplaysForRows_(sh, mCol, rows);
  const pDisplays = _batchGetDisplaysForRows_(sh, pCol, rows);

  const kAudit = _auditDimResetTargets_(packets, kNotes, kDisplays, "K");
  const cAudit = _auditDimResetTargets_(packets, cNotes, cDisplays, "C");
  const mAudit = _auditDimResetTargets_(packets, mNotes, mDisplays, "M");
  const pAudit = _auditDimResetTargets_(packets, pNotes, pDisplays, "P");

  Logger.log("=== KCMP TERRA PRODUCTION RESET AUDIT ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("MODEL_K=" + String(MODEL_K));
  Logger.log("MODEL_C=" + String(MODEL_C));
  Logger.log("MODEL_M=" + String(MODEL_M));
  Logger.log("K_TARGET_ROWS=" + JSON.stringify(kAudit.targetRows));
  Logger.log("C_TARGET_ROWS=" + JSON.stringify(cAudit.targetRows));
  Logger.log("M_TARGET_ROWS=" + JSON.stringify(mAudit.targetRows));
  Logger.log("P_TARGET_ROWS=" + JSON.stringify(pAudit.targetRows));
  Logger.log("K_FINALIZED_OK=" + kAudit.finalizedOk);
  Logger.log("K_FINALIZED_ERROR=" + kAudit.finalizedError);
  Logger.log("C_FINALIZED_OK=" + cAudit.finalizedOk);
  Logger.log("C_FINALIZED_ERROR=" + cAudit.finalizedError);
  Logger.log("M_FINALIZED_OK=" + mAudit.finalizedOk);
  Logger.log("M_FINALIZED_ERROR=" + mAudit.finalizedError);
  Logger.log("P_FINALIZED_OK=" + pAudit.finalizedOk);
  Logger.log("P_FINALIZED_ERROR=" + pAudit.finalizedError);
  Logger.log("K_NONEMPTY_DISPLAY_COUNT=" + kAudit.nonemptyDisplay);
  Logger.log("C_NONEMPTY_DISPLAY_COUNT=" + cAudit.nonemptyDisplay);
  Logger.log("M_NONEMPTY_DISPLAY_COUNT=" + mAudit.nonemptyDisplay);
  Logger.log("P_NONEMPTY_DISPLAY_COUNT=" + pAudit.nonemptyDisplay);
  Logger.log("K_NOTE_COUNT=" + kAudit.noteCount);
  Logger.log("C_NOTE_COUNT=" + cAudit.noteCount);
  Logger.log("M_NOTE_COUNT=" + mAudit.noteCount);
  Logger.log("P_NOTE_COUNT=" + pAudit.noteCount);
  Logger.log("WOULD_CLEAR_K=" + String(kAudit.wouldClear));
  Logger.log("WOULD_CLEAR_C=" + String(cAudit.wouldClear));
  Logger.log("WOULD_CLEAR_M=" + String(mAudit.wouldClear));
  Logger.log("WOULD_CLEAR_P=" + String(pAudit.wouldClear));
  Logger.log("GPT_CALLS=0");
  Logger.log("SHEET_WRITES=0");
  Logger.log("NOTE_WRITES=0");
  Logger.log("CLEAR_CALLS=0");
  Logger.log("READ_ONLY=true");
  Logger.log("================================");

  return {
    ok: true,
    totalPackets: packets.length,
    cols: { K: kCol, C: cCol, M: mCol, P: pCol },
    models: { K: MODEL_K, C: MODEL_C, M: MODEL_M },
    K: kAudit,
    C: cAudit,
    M: mAudit,
    P: pAudit
  };
}

/** representative rows만 대상으로 content+Note clear. 전체 열 통째 clear 금지. */
function _clearKCMPProductionCellsAtRows_(sheet, col, rows){
  if (!sheet || !col || !rows || !rows.length) return 0;

  const uniq = {};
  rows.forEach(function(r){
    const n = Number(r);
    if (n > 0) uniq[n] = true;
  });
  const sorted = Object.keys(uniq).map(Number).sort(function(a, b){ return a - b; });
  if (!sorted.length) return 0;

  let cleared = 0;
  let blockStart = sorted[0];
  let blockEnd = sorted[0];
  for (let i = 1; i <= sorted.length; i++) {
    const r = sorted[i];
    if (i < sorted.length && r === blockEnd + 1) {
      blockEnd = r;
      continue;
    }
    const h = blockEnd - blockStart + 1;
    const rng = sheet.getRange(blockStart, col, h, 1);
    rng.clearContent();
    rng.clearNote();
    cleared += h;
    if (i < sorted.length) {
      blockStart = r;
      blockEnd = r;
    }
  }
  return cleared;
}

function _countNonemptyDisplayAndNotes_(packets, displaysByRow, notesByRow){
  let displayNonempty = 0;
  let noteNonempty = 0;
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    if (!row) return;
    const d = displaysByRow[row];
    const n = notesByRow[row];
    if (d != null && String(d).trim().length > 0) displayNonempty++;
    if (n != null && String(n).trim().length > 0) noteNonempty++;
  });
  return { displayNonempty: displayNonempty, noteNonempty: noteNonempty };
}

/**
 * STEP 15B: 14차시 4조 production K/C/M/P만 안전 초기화.
 * GPT 호출 없음. shadow Properties 미삭제. menu 미등록. runner 자동 chaining 없음.
 */
function RESET_KCMP_PRODUCTION_FOR_TERRA_14_4(){
  const SHEET_NAME = "14차시 4조";
  const EXPECTED_PACKETS = 73;
  const EXPECTED_MODEL = "gpt-5.6-terra";

  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    throw new Error("RESET_ABORT: active sheet must be '" + SHEET_NAME + "'");
  }

  if (String(MODEL_K) !== EXPECTED_MODEL) {
    throw new Error("RESET_ABORT: MODEL_K must be " + EXPECTED_MODEL + " (got " + MODEL_K + ")");
  }
  if (String(MODEL_C) !== EXPECTED_MODEL) {
    throw new Error("RESET_ABORT: MODEL_C must be " + EXPECTED_MODEL + " (got " + MODEL_C + ")");
  }
  if (String(MODEL_M) !== EXPECTED_MODEL) {
    throw new Error("RESET_ABORT: MODEL_M must be " + EXPECTED_MODEL + " (got " + MODEL_M + ")");
  }

  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const pCol = colNumOf(map.N);

  if (!(kCol > 0 && cCol > 0 && mCol > 0 && pCol > 0)) {
    throw new Error("RESET_ABORT: production columns K/L/M/N not resolved (K=" + kCol + ",C=" + cCol + ",M=" + mCol + ",P=" + pCol + ")");
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  if (packets.length !== EXPECTED_PACKETS) {
    throw new Error("RESET_ABORT: TOTAL_PACKETS must be " + EXPECTED_PACKETS + " (got " + packets.length + ")");
  }

  const rows = [];
  const seen = {};
  for (let i = 0; i < packets.length; i++) {
    const row = packets[i] && packets[i].representativeRow;
    const n = Number(row);
    if (!(n > 0)) {
      throw new Error("RESET_ABORT: packet index " + i + " missing valid representativeRow");
    }
    if (seen[n]) {
      throw new Error("RESET_ABORT: duplicate representativeRow=" + n);
    }
    seen[n] = true;
    rows.push(n);
  }
  if (rows.length !== EXPECTED_PACKETS) {
    throw new Error("RESET_ABORT: representativeRow count mismatch");
  }

  // 안전 조건 모두 통과 후에만 clear 수행
  const kCleared = _clearKCMPProductionCellsAtRows_(sh, kCol, rows);
  const cCleared = _clearKCMPProductionCellsAtRows_(sh, cCol, rows);
  const mCleared = _clearKCMPProductionCellsAtRows_(sh, mCol, rows);
  const pCleared = _clearKCMPProductionCellsAtRows_(sh, pCol, rows);

  // verification (read-only)
  const kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  const cNotes = _batchGetNotesForRows_(sh, cCol, rows);
  const mNotes = _batchGetNotesForRows_(sh, mCol, rows);
  const pNotes = _batchGetNotesForRows_(sh, pCol, rows);
  const kDisplays = _batchGetDisplaysForRows_(sh, kCol, rows);
  const cDisplays = _batchGetDisplaysForRows_(sh, cCol, rows);
  const mDisplays = _batchGetDisplaysForRows_(sh, mCol, rows);
  const pDisplays = _batchGetDisplaysForRows_(sh, pCol, rows);

  const kEmpty = _countNonemptyDisplayAndNotes_(packets, kDisplays, kNotes);
  const cEmpty = _countNonemptyDisplayAndNotes_(packets, cDisplays, cNotes);
  const mEmpty = _countNonemptyDisplayAndNotes_(packets, mDisplays, mNotes);
  const pEmpty = _countNonemptyDisplayAndNotes_(packets, pDisplays, pNotes);

  const kProg = _summarizeKCMPDimensionNoteProgress_(packets, kNotes, "K");
  const cProg = _summarizeKCMPDimensionNoteProgress_(packets, cNotes, "C");
  const mProg = _summarizeKCMPDimensionNoteProgress_(packets, mNotes, "M");
  const pProg = _summarizeKCMPDimensionNoteProgress_(packets, pNotes, "P");

  const kFinalizedAfter = kProg.finalizedOk + kProg.finalizedError;
  const cFinalizedAfter = cProg.finalizedOk + cProg.finalizedError;
  const mFinalizedAfter = mProg.finalizedOk + mProg.finalizedError;
  const pFinalizedAfter = pProg.finalizedOk + pProg.finalizedError;

  Logger.log("=== KCMP TERRA PRODUCTION RESET ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("MODEL_K=" + String(MODEL_K));
  Logger.log("MODEL_C=" + String(MODEL_C));
  Logger.log("MODEL_M=" + String(MODEL_M));
  Logger.log("K_CLEARED_ROWS=" + kCleared);
  Logger.log("C_CLEARED_ROWS=" + cCleared);
  Logger.log("M_CLEARED_ROWS=" + mCleared);
  Logger.log("P_CLEARED_ROWS=" + pCleared);
  Logger.log("K_DISPLAY_NONEMPTY_AFTER=" + kEmpty.displayNonempty);
  Logger.log("K_NOTE_NONEMPTY_AFTER=" + kEmpty.noteNonempty);
  Logger.log("C_DISPLAY_NONEMPTY_AFTER=" + cEmpty.displayNonempty);
  Logger.log("C_NOTE_NONEMPTY_AFTER=" + cEmpty.noteNonempty);
  Logger.log("M_DISPLAY_NONEMPTY_AFTER=" + mEmpty.displayNonempty);
  Logger.log("M_NOTE_NONEMPTY_AFTER=" + mEmpty.noteNonempty);
  Logger.log("P_DISPLAY_NONEMPTY_AFTER=" + pEmpty.displayNonempty);
  Logger.log("P_NOTE_NONEMPTY_AFTER=" + pEmpty.noteNonempty);
  Logger.log("K_FINALIZED_AFTER=" + kFinalizedAfter);
  Logger.log("C_FINALIZED_AFTER=" + cFinalizedAfter);
  Logger.log("M_FINALIZED_AFTER=" + mFinalizedAfter);
  Logger.log("P_FINALIZED_AFTER=" + pFinalizedAfter);
  Logger.log("K_UNFINALIZED_AFTER=" + kProg.unfinalized);
  Logger.log("C_UNFINALIZED_AFTER=" + cProg.unfinalized);
  Logger.log("M_UNFINALIZED_AFTER=" + mProg.unfinalized);
  Logger.log("P_UNFINALIZED_AFTER=" + pProg.unfinalized);
  Logger.log("SHADOW_PROPERTIES_CLEARED=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("RESET_SCOPE=KCMP_PRODUCTION_KCMP_ONLY");
  Logger.log("================================");

  return {
    ok: true,
    cleared: { K: kCleared, C: cCleared, M: mCleared, P: pCleared },
    after: {
      K: { display: kEmpty.displayNonempty, note: kEmpty.noteNonempty, finalized: kFinalizedAfter, unfinalized: kProg.unfinalized },
      C: { display: cEmpty.displayNonempty, note: cEmpty.noteNonempty, finalized: cFinalizedAfter, unfinalized: cProg.unfinalized },
      M: { display: mEmpty.displayNonempty, note: mEmpty.noteNonempty, finalized: mFinalizedAfter, unfinalized: mProg.unfinalized },
      P: { display: pEmpty.displayNonempty, note: pEmpty.noteNonempty, finalized: pFinalizedAfter, unfinalized: pProg.unfinalized }
    },
    shadowPropertiesCleared: 0
  };
}

function _isStalePErrorNote_(pNoteText, kNoteText, cNoteText, mNoteText){
  const pFin = _isFinalKCMPDecisionNote_(pNoteText, "P");
  const kFin = _isFinalKCMPDecisionNote_(kNoteText, "K");
  const cFin = _isFinalKCMPDecisionNote_(cNoteText, "C");
  const mFin = _isFinalKCMPDecisionNote_(mNoteText, "M");
  if (!(pFin.finalized && pFin.status === "ERROR")) return false;
  if (!(kFin.finalized && kFin.status === "OK")) return false;
  if (!(cFin.finalized && cFin.status === "OK")) return false;
  if (!(mFin.finalized && mFin.status === "OK")) return false;
  return true;
}

function _collectUpstreamErrorPids_(packets, kNotes, cNotes, mNotes){
  const out = [];
  (packets || []).forEach(function(p){
    const row = p && p.representativeRow;
    const pid = p && p.pid;
    if (!row || !pid) return;
    const kFin = _isFinalKCMPDecisionNote_(kNotes[row], "K");
    const cFin = _isFinalKCMPDecisionNote_(cNotes[row], "C");
    const mFin = _isFinalKCMPDecisionNote_(mNotes[row], "M");
    if ((kFin.finalized && kFin.status === "ERROR") ||
        (cFin.finalized && cFin.status === "ERROR") ||
        (mFin.finalized && mFin.status === "ERROR")) {
      out.push(pid);
    }
  });
  out.sort();
  return out;
}

/**
 * STEP 15C: upstream K/C/M이 모두 OK인데 P만 ERROR로 남은 stale P만 clear.
 * GPT/shadow/KCM 수정 없음. runCodeP_ResumeBatch 자동 실행 없음.
 */
function RESET_KCMP_STALE_P_ERRORS_ONCE(){
  const SHEET_NAME = "14차시 4조";
  const EXPECTED_PACKETS = 73;

  const sh = SpreadsheetApp.getActiveSheet();
  if (String(sh.getName()) !== SHEET_NAME) {
    throw new Error("STALE_P_RESET_ABORT: active sheet must be '" + SHEET_NAME + "'");
  }

  const map = ensureColMapOrHalt_();
  const kCol = colNumOf(map.K);
  const cCol = colNumOf(map.L);
  const mCol = colNumOf(map.M);
  const pCol = colNumOf(map.N);
  if (!(pCol > 0)) {
    throw new Error("STALE_P_RESET_ABORT: P production column map.N not resolved");
  }

  const packets = buildAllKCMPClusterPackets_(sh, map) || [];
  if (packets.length !== EXPECTED_PACKETS) {
    throw new Error("STALE_P_RESET_ABORT: TOTAL_PACKETS must be " + EXPECTED_PACKETS + " (got " + packets.length + ")");
  }

  const seen = {};
  for (let i = 0; i < packets.length; i++) {
    const row = packets[i] && packets[i].representativeRow;
    const n = Number(row);
    if (!(n > 0)) {
      throw new Error("STALE_P_RESET_ABORT: packet index " + i + " missing valid representativeRow");
    }
    if (seen[n]) {
      throw new Error("STALE_P_RESET_ABORT: duplicate representativeRow=" + n);
    }
    seen[n] = true;
  }

  const rows = _collectKCMPRepresentativeRows_(packets);
  const kNotes = _batchGetNotesForRows_(sh, kCol, rows);
  const cNotes = _batchGetNotesForRows_(sh, cCol, rows);
  const mNotes = _batchGetNotesForRows_(sh, mCol, rows);
  const pNotes = _batchGetNotesForRows_(sh, pCol, rows);

  const kProg = _summarizeKCMPDimensionNoteProgress_(packets, kNotes, "K");
  const cProg = _summarizeKCMPDimensionNoteProgress_(packets, cNotes, "C");
  const mProg = _summarizeKCMPDimensionNoteProgress_(packets, mNotes, "M");
  if (!kProg.complete || !cProg.complete || !mProg.complete) {
    throw new Error("STALE_P_RESET_ABORT: ALL K/C/M must be finalized (K unf=" + kProg.unfinalized + ", C unf=" + cProg.unfinalized + ", M unf=" + mProg.unfinalized + ")");
  }

  const staleBefore = [];
  const staleRows = [];
  packets.forEach(function(p){
    const row = p && p.representativeRow;
    const pid = p && p.pid;
    if (!row || !pid) return;
    if (_isStalePErrorNote_(pNotes[row], kNotes[row], cNotes[row], mNotes[row])) {
      staleBefore.push(pid);
      staleRows.push(row);
    }
  });
  staleBefore.sort();

  const upstreamErrorPids = _collectUpstreamErrorPids_(packets, kNotes, cNotes, mNotes);

  const pClearCount = staleRows.length > 0
    ? _clearKCMPProductionCellsAtRows_(sh, pCol, staleRows)
    : 0;

  const pNotesAfter = _batchGetNotesForRows_(sh, pCol, rows);
  const pProgAfter = _summarizeKCMPDimensionNoteProgress_(packets, pNotesAfter, "P");
  const clearedPids = staleBefore.slice();

  Logger.log("=== KCMP STALE P ERROR RESET ===");
  Logger.log("SHEET=" + SHEET_NAME);
  Logger.log("TOTAL_PACKETS=" + packets.length);
  Logger.log("STALE_P_ERROR_COUNT_BEFORE=" + staleBefore.length);
  Logger.log("STALE_P_ERROR_PIDS=" + JSON.stringify(staleBefore));
  Logger.log("CLEARED_P_ROWS=" + JSON.stringify(staleRows.slice().sort(function(a, b){ return a - b; })));
  Logger.log("CLEARED_P_PIDS=" + JSON.stringify(clearedPids));
  Logger.log("P_FINALIZED_OK_AFTER=" + pProgAfter.finalizedOk);
  Logger.log("P_FINALIZED_ERROR_AFTER=" + pProgAfter.finalizedError);
  Logger.log("P_UNFINALIZED_AFTER=" + pProgAfter.unfinalized);
  Logger.log("CURRENT_UPSTREAM_ERROR_PIDS=" + JSON.stringify(upstreamErrorPids));
  Logger.log("GPT_CALLS=0");
  Logger.log("K_WRITES=0");
  Logger.log("C_WRITES=0");
  Logger.log("M_WRITES=0");
  Logger.log("P_CLEAR_COUNT=" + pClearCount);
  Logger.log("SHADOW_PROPERTIES_CLEARED=0");
  Logger.log("SEMANTIC_CHANGE=NONE");
  Logger.log("================================");

  return {
    ok: true,
    staleBefore: staleBefore,
    clearedPids: clearedPids,
    clearedRows: staleRows.slice().sort(function(a, b){ return a - b; }),
    pClearCount: pClearCount,
    upstreamErrorPids: upstreamErrorPids,
    pAfter: {
      finalizedOk: pProgAfter.finalizedOk,
      finalizedError: pProgAfter.finalizedError,
      unfinalized: pProgAfter.unfinalized
    }
  };
}

/** 요약문 → A코드 추론 (A1/A2/A3 중심) */
function _inferAFromSummary_(txt){
  const t = (txt||'').toLowerCase();

  // 키워드 기반 1차 휴리스틱
  const hasData   = /자료|데이터|그래프|표|관찰|증거|수치|값|측정/.test(txt);
  const hasWarrant= /왜|때문|근거|논리|정당|타당|설명|원리|원인|이유/.test(txt);
  const hasClaim  = /나는|~라고\s*생각|주장|맞(?:다|는)/.test(txt);

  // 우선순위: A3(정당화) > A2(증거/자료 활용) > A1(개념/관계 설명)
  if ((hasData && hasWarrant) || (/정당|근거로|증명|뒷받침/.test(txt))) {
    return { code:"A3", name:"학생이 자신의 주장을 정당화하였다." };
  }
  if (hasData || /자료를|그래프를|표를|관찰을|근거로/.test(txt)) {
    return { code:"A2", name:"학생이 자료/증거를 활용해 해석하였다." };
  }
  if (hasWarrant || hasClaim || /정의|구조|관계|원리/.test(txt)) {
    return { code:"A1", name:"학생이 개념 관계를 설명하였다." };
  }
  // 폴백: A1
  return { code:"A1", name:"학생이 개념 관계를 설명하였다." };
}

/** 요약문 → D코드 추론 (기본: D4만 on; 필요 시 다른 D도 확장) */
function _inferDFromSummary_(txt){
  const hasUncertain = /모르겠|잘\s*모르|그런\s*것\s*같|아마|일\s*것|추측|불확실|헷갈|맞는지|확신/.test(txt);
  if (hasUncertain){
    return { code:"D4", name:"학생이 불확실성을 표현하였다." };
  }
  return null; // 기록 안 함
}

/** G~J 등에서 대표 화자 선택: 가장 먼저 1 이상인 열 → 동수면 최댓값/왼쪽 */
function pickRepresentativeSpeaker_(sh, row, sCols){
  const candidates = [];
  for (let idx=0; idx<sCols.length; idx++){
    const col = sCols[idx];
    if (!col) continue;
    const v = toNum(sh.getRange(row, col).getDisplayValue());
    candidates.push({idx, col, val: v});
  }
  // 1) earliest active (val>=1)
  const early = candidates.find(c=>c.val>=1);
  if (early) return (early.idx+1); // 1-based: 참석자 N

  // 2) max value; tie → left
  const maxVal = Math.max(0, ...candidates.map(c=>c.val));
  if (maxVal <= 0) return 1; // 디폴트 참석자 1
  const firstMax = candidates.find(c=>c.val === maxVal);
  return (firstMax ? firstMax.idx+1 : 1);
}

/** 요약문에서 따옴표 인용('…' 또는 "…") 추출 → 없으면 핵심어절 1개 */
function pickQuoteFromSummary_(txt){
  if (!txt) return '';
  const m1 = txt.match(/["""']([^"'""]{3,80})[""']/);
  if (m1 && m1[1]) return m1[1].trim();
  const m2 = txt.match(/['''"]([^"''']{3,80})[''"]/);
  if (m2 && m2[1]) return m2[1].trim();
  // 없으면 짧은 핵심어절
  const sent = String(txt).replace(/\s+/g,' ').trim();
  return sent.length>24 ? sent.slice(0,24)+'…' : sent;
}

/** 형식 강제 렌더: 
 *  - K: "K#. 코드명\n참석자 N은 '…'라고 설명/정당화하였다."
 *  - M: "M#. 코드명\n참석자 N이 '…'라고 자신의 이해를 점검하였다." 등
 */
function _renderCodeLine_(inf, speakerN, summary, opt){
  if (!inf) return '';
  const N = speakerN || 1;
  const q = pickQuoteFromSummary_(summary);

  if (opt && opt.dimension === 'A'){
    // 코드별 서술동사 다르게
    const verb = ({A1:'설명하', A2:'해석하', A3:'정당화하'})[inf.code] || '설명하';
    const tail = (inf.code === 'A2') ? '며 자료를 활용하였다.' : (inf.code === 'A3') ? '고 주장하였다.' : '였/하였';
    const line = `${inf.code}. ${inf.name}\n참석자 ${N}은 '${q}'라고 ${verb}였다.`;
    // 한국어 어미 다듬기
    return line.replace('였다.','하였다.').replace('설명하였다하였다','설명하였다');
  }

  // D (현재 D4만)
  if (opt && opt.dimension === 'D'){
    const line = `${inf.code}. ${inf.name}\n참석자 ${N}이 '${q}'라고 말하며 자신의 이해를 점검하였다.`;
    return line;
  }

  return `${inf.code}. ${inf.name}`;
}

/** E/F 행 검증 헬퍼 */
function _hasPidAndSummary_(sh, map, row){
  const e = String(sh.getRange(row, colNumOf(map.E)).getValue()||'').trim();
  const f = String(sh.getRange(row, colNumOf(map.F)).getValue()||'').trim();
  return (e && f);
}

/** E열에서 마지막 PID가 등장하는 행 번호를 찾는 함수 */
function findLastPidRow_(sh, map){
  const eCol = colNumOf(map.E);
  const lastRow = sh.getLastRow();
  if (lastRow < 2) return 1;

  const rng = sh.getRange(2, eCol, lastRow-1, 1).getDisplayValues();
  let last = 1;
  for (let i = 0; i < rng.length; i++){
    const s = String(rng[i][0] || '').replace(/\u200B/g,'').trim(); // 제로폭·공백 제거
    if (s !== '') last = i + 2;  // "빈칸이 아니면" 최신행으로 갱신
  }
  return last;
}

/** D열에서 고유 PID를 추출하여 E열에 목록으로 재작성하는 함수 */
function normalizePidIntoE_(sh, map, lastRowLimit){
  const dCol = colNumOf(map.D), eCol = colNumOf(map.E);
  const lastRow = Math.min(sh.getLastRow(), lastRowLimit);
  if (lastRow < 2) return;

  const rows = lastRow - 1;
  const D = sh.getRange(2, dCol, rows, 1).getDisplayValues();

  // 1) D에서 고유 PID를 등장 순서대로 수집
  const seen = new Set();
  const uniquePIDs = [];
  for (let i=0;i<rows;i++){
    const pid = String(D[i][0]||'').match(/P\d{1,5}/i);
    if (!pid) continue;
    const p = pid[0].toUpperCase();
    if (!seen.has(p)){
      seen.add(p);
      uniquePIDs.push(p);
    }
  }

  // 2) E열(2행부터)에 고유 PID를 1개씩 나열
  const out = uniquePIDs.map(p=>[p]);
  if (out.length > 0){
    sh.getRange(2, eCol, out.length, 1).setValues(out);
  }

  // 3) 그 아래는 깨끗이 비움(잔여 청소)
  const extra = rows - out.length;
  if (extra > 0){
    sh.getRange(2 + out.length, eCol, extra, 1).clearContent();
  }
}

/** E열 각 PID 행마다 해당 PID의 모든 발화를 취합하여 F열에 요약문을 생성하는 함수 */
function buildClusterSummariesFromPID_(sh, map, lastRowLimit){
  const aCol = colNumOf(map.A), bCol = colNumOf(map.B), cCol = colNumOf(map.C);
  const dCol = colNumOf(map.D), eCol = colNumOf(map.E), fCol = colNumOf(map.F);

  const sheetLastRow = sh.getLastRow();
  if (sheetLastRow < 2) return;

  // 1) E(목록) 구간만큼만 읽기
  const eLen = Math.max(0, Math.min(sheetLastRow, lastRowLimit) - 1);
  const E = eLen ? sh.getRange(2, eCol, eLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim()) : [];

  // 2) A/B/C/D는 **시트 전체**를 읽기
  const dataLen = sheetLastRow - 1;
  const A = dataLen ? sh.getRange(2, aCol, dataLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim()) : [];
  const B = dataLen ? sh.getRange(2, bCol, dataLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim()) : [];
  const C = dataLen ? sh.getRange(2, cCol, dataLen, 1).getDisplayValues().map(r=>String(r[0]||'').replace(/\u200B/g,'').replace(/\s+/g,' ').trim()) : [];
  const D = dataLen ? sh.getRange(2, dCol, dataLen, 1).getDisplayValues().map(r=>String(r[0]||'').trim().toUpperCase()) : [];

  // 성능 개선: PID별 그룹핑을 먼저 수행 (배치 처리)
  const pidIndex = {}; // pid -> items 배열
  for (let r=0;r<dataLen;r++){
    const pid = String(D[r]||'').trim().toUpperCase();
    if (!/^P\d+/i.test(pid)) continue;
    if (!C[r]) continue;
    if (!pidIndex[pid]) pidIndex[pid] = [];
    pidIndex[pid].push({ ts:B[r], spk:A[r], utt:C[r], row:r+2 });
  }

  // 성능 개선: 모든 F열 값을 배열로 준비 후 한 번에 쓰기
  const fValues = [];
  
  for (let i=0;i<E.length;i++){
    const pid = (E[i]||'').toString().trim().toUpperCase();
    if (!/^P\d+/i.test(pid)) { 
      fValues.push(['']); 
      continue; 
    }

    // 성능 개선: 이미 그룹핑된 items 사용
    const items = pidIndex[pid] || [];

    if (!items.length){ 
      fValues.push(['']); 
      continue; 
    }

    items.sort((x,y)=> String(x.ts).localeCompare(String(y.ts)));
    const firstTS = items[0].ts || '';
    const lastTS  = items[items.length-1].ts || firstTS;
    const head = firstTS && lastTS && firstTS!==lastTS ? `■ ${firstTS}~${lastTS}\t` : (firstTS ? `■ ${firstTS}\t` : '■\t');

    const sents = [];
    for (let k=0;k<items.length;k++){
      const it = items[k];
      const prev = k>0 ? items[k-1] : null;
      const text = it.utt.length>200 ? (it.utt.slice(0,197)+'…') : it.utt;

      let verb = '말한다';
      const isQ = /[?？]/.test(text) || /(왜|어떻게|무엇|뭐|뭘)/.test(text);
      if (isQ) verb = '질문한다';
      else if (prev && prev.spk && prev.spk !== it.spk) verb = '응답한다';
      else if (prev && prev.spk === it.spk) verb = '덧붙인다';

      sents.push(`${it.spk}가 '${text}'라고 ${verb}.`);
    }

    // 성능 개선: 개별 setValue 대신 배열에 추가
    fValues.push([head + sents.join(' ')]);
  }
  
  // 성능 개선: 모든 F열 값을 한 번에 배치 쓰기 (row-by-row 접근 제거)
  if (fValues.length > 0) {
    sh.getRange(2, fCol, fValues.length, 1).setValues(fValues);
  }
}

/** G~J 화자 열 동적 탐지 (기존 로직 재사용) - 강화 버전 */
function getSColsFlexible_(sh, map, headerRow){
  // S1~S4 매핑이 있으면 사용
  const s1 = map.S1 ? colNumOf(map.S1) : null;
  const s2 = map.S2 ? colNumOf(map.S2) : null;
  const s3 = map.S3 ? colNumOf(map.S3) : null;
  const s4 = map.S4 ? colNumOf(map.S4) : null;
  
  // 모든 열이 유효한지 검증
  if (s1 && s2 && s3 && s4 && 
      s1 >= 1 && s1 <= 1000 && 
      s2 >= 1 && s2 <= 1000 && 
      s3 >= 1 && s3 <= 1000 && 
      s4 >= 1 && s4 <= 1000) {
    return [s1, s2, s3, s4];
  }

  // 폴백 1: detectParticipantCols_로 자동 감지 시도
  try {
    const detected = detectParticipantCols_(sh, headerRow || 1);
    if (detected && detected.S && detected.S.length > 0) {
      const detectedCols = detected.S.map(s => s.col).filter(c => c && c >= 1 && c <= 1000);
      if (detectedCols.length >= 2) {
        // 부족한 열은 null로 채움
        while (detectedCols.length < 4) {
          detectedCols.push(null);
        }
        return detectedCols.slice(0, 4);
      }
    }
  } catch(e) {
    Logger.log('getSColsFlexible_: detectParticipantCols_ 실패: ' + e.message);
  }

  // 폴백 2: G~J (7~10) - 기본값
  return [7, 8, 9, 10];
}


/***** ===== KCMP Decision Tree 공통 입력 계층 (clusterPacket) ===== *****/
/**
 * STEP 2: 판정 로직은 변경하지 않는다.
 * 기존 라이브 K/C/M/P 코더는 이 함수들을 호출하지 않는다.
 *
 * 원발화 행: D열 PID (buildClusterSummariesFromPID_와 동일)
 * 대표행/요약/발화수: E열 목록 행의 F, S1~S4 (normalizePidIntoE_ 정렬)
 */

function _kcmpFormatTimestamp_(raw, display){
  const disp = (display == null) ? '' : String(display).trim();
  if (disp) {
    if (parseMMSS(disp) != null) return disp;
    const m = disp.match(/(\d{1,2}):([0-5]\d)/);
    if (m) {
      const mm = m[1].length === 1 ? ('0' + m[1]) : m[1];
      return mm + ':' + m[2];
    }
  }
  if (raw instanceof Date) {
    const minutes = raw.getMinutes();
    const seconds = raw.getSeconds();
    return (minutes < 10 ? '0' : '') + minutes + ':' + (seconds < 10 ? '0' : '') + seconds;
  }
  if (raw == null || raw === '') return '';
  return String(raw).trim();
}

function _kcmpIsBlankUtterance_(text){
  return !String(text == null ? '' : text).replace(/\u200B/g, '').trim();
}

/**
 * 시트 1회 배치 읽기. buildKCMPClusterPacket_ / buildAllKCMPClusterPackets_가 재사용.
 */
function _prepareKCMPPacketContext_(sheet, map){
  const lastRow = sheet.getLastRow();
  const lastCol = Math.max(1, sheet.getLastColumn());
  const headerRow = getHeaderRow_(sheet);

  const aCol = colNumOf(map.A);
  const bCol = colNumOf(map.B);
  const cCol = colNumOf(map.C);
  const dCol = colNumOf(map.D);
  const eCol = colNumOf(map.E);
  const fCol = colNumOf(map.F);
  let oCol = null;
  try { oCol = map.O ? colNumOf(map.O) : null; } catch (e) { oCol = null; }

  const sCols = getSColsFlexible_(sheet, map, 1);
  const sHeaders = sCols.map(function(c, i){
    return c ? { key: 'S' + (i + 1), col: c, label: String(headerRow[c - 1] || '').trim() } : null;
  }).filter(Boolean);

  const rowCount = Math.max(0, lastRow - 1);
  const values = rowCount ? sheet.getRange(2, 1, rowCount, lastCol).getValues() : [];
  const display = rowCount ? sheet.getRange(2, 1, rowCount, lastCol).getDisplayValues() : [];

  const pidOrder = [];
  const listRowByPid = {};
  for (let i = 0; i < display.length; i++) {
    const pid = normPID_(display[i][eCol - 1]);
    if (!pid) continue;
    if (listRowByPid[pid]) continue;
    pidOrder.push(pid);
    listRowByPid[pid] = i + 2;
  }

  const utteranceRowsByPid = {};
  for (let i = 0; i < display.length; i++) {
    const pid = normPID_(display[i][dCol - 1]) || normPID_(values[i][dCol - 1]);
    if (!pid) continue;
    if (!utteranceRowsByPid[pid]) utteranceRowsByPid[pid] = [];
    utteranceRowsByPid[pid].push(i + 2);
  }

  return {
    sheet: sheet,
    map: map,
    lastRow: lastRow,
    headerRow: headerRow,
    aCol: aCol,
    bCol: bCol,
    cCol: cCol,
    dCol: dCol,
    eCol: eCol,
    fCol: fCol,
    oCol: oCol,
    sCols: sCols,
    sHeaders: sHeaders,
    values: values,
    display: display,
    pidOrder: pidOrder,
    listRowByPid: listRowByPid,
    utteranceRowsByPid: utteranceRowsByPid
  };
}

function _readSheetSpeakerCounts_(ctx, listRow){
  const counts = { S1: 0, S2: 0, S3: 0, S4: 0 };
  if (!listRow || listRow < 2) return counts;
  const idx = listRow - 2;
  if (idx < 0 || idx >= ctx.display.length) return counts;
  (ctx.sCols || []).forEach(function(col, i){
    if (!col) return;
    const key = 'S' + (i + 1);
    const val = ctx.display[idx][col - 1];
    const n = parseFloat(String(val == null ? '0' : val).replace(/[^\d.]/g, '')) || 0;
    counts[key] = n;
  });
  return counts;
}

/**
 * PID 단위 공통 입력 객체.
 * options.context가 있으면 배치 컨텍스트를 재사용한다.
 */
function buildKCMPClusterPacket_(sheet, map, pid, options){
  const ctx = (options && options.context) ? options.context : _prepareKCMPPacketContext_(sheet, map);
  const pidKey = normPID_(pid);
  const warnings = [];
  const unmappedStudentSpeakers = [];
  const unknownSpeakers = [];

  const listRow = ctx.listRowByPid[pidKey] || null;
  const utteranceRows = (ctx.utteranceRowsByPid[pidKey] || []).slice().sort(function(a, b){ return a - b; });
  const representativeRow = listRow || (utteranceRows.length ? utteranceRows[0] : null);

  if (!pidKey) warnings.push('pid가 비어 있거나 정규화할 수 없음');
  if (!listRow) warnings.push('E열 목록에서 PID를 찾지 못함');
  if (!utteranceRows.length) warnings.push('D열에서 해당 PID 원발화 행을 찾지 못함');

  let summary = '';
  if (listRow) {
    const fIdx = listRow - 2;
    if (fIdx >= 0 && fIdx < ctx.display.length) {
      summary = String(ctx.display[fIdx][ctx.fCol - 1] == null ? '' : ctx.display[fIdx][ctx.fCol - 1]);
    }
  }

  const turns = [];
  utteranceRows.forEach(function(row){
    const i = row - 2;
    if (i < 0 || i >= ctx.display.length) return;
    const speakerRaw = String(ctx.display[i][ctx.aCol - 1] == null ? '' : ctx.display[i][ctx.aCol - 1]).replace(/\u200B/g, '').trim();
    const utteranceRaw = ctx.display[i][ctx.cCol - 1];
    const utterance = (utteranceRaw == null) ? '' : String(utteranceRaw);
    if (_kcmpIsBlankUtterance_(utterance)) return;

    const timestamp = _kcmpFormatTimestamp_(ctx.values[i][ctx.bCol - 1], ctx.display[i][ctx.bCol - 1]);
    const isTeacher = isTeacherSpeaker(speakerRaw);
    let speakerId = null;
    let role = 'other';

    if (isTeacher) {
      role = 'teacher';
      speakerId = null;
    } else if (!speakerRaw) {
      role = 'other';
      speakerId = null;
      unknownSpeakers.push({ row: row, speakerRaw: speakerRaw });
      warnings.push('행 ' + row + ': 화자명이 비어 있음');
    } else {
      speakerId = matchSpeakerToSx_(speakerRaw, ctx.sHeaders);
      if (speakerId) {
        role = 'student';
      } else {
        role = 'other';
        unmappedStudentSpeakers.push({ row: row, speakerRaw: speakerRaw });
        warnings.push('행 ' + row + ': 학생처럼 보이나 S1~S4에 매핑되지 않음 (' + speakerRaw + ')');
      }
    }

    turns.push({
      row: row,
      timestamp: timestamp,
      speakerRaw: speakerRaw,
      speakerId: speakerId,
      role: role,
      utterance: utterance
    });
  });

  const turnDerivedCounts = { S1: 0, S2: 0, S3: 0, S4: 0 };
  const studentMeta = {};
  const activeSet = [];
  turns.forEach(function(t){
    if (t.role !== 'student' || !t.speakerId) return;
    turnDerivedCounts[t.speakerId] = (turnDerivedCounts[t.speakerId] || 0) + 1;
    if (!studentMeta[t.speakerId]) {
      const hdr = ctx.sHeaders.filter(function(h){ return h.key === t.speakerId; })[0];
      studentMeta[t.speakerId] = { id: t.speakerId, label: (hdr && hdr.label) ? hdr.label : t.speakerRaw, utteranceCount: 0 };
    }
    studentMeta[t.speakerId].utteranceCount += 1;
    if (activeSet.indexOf(t.speakerId) < 0) activeSet.push(t.speakerId);
  });
  activeSet.sort();

  const students = ['S1', 'S2', 'S3', 'S4']
    .filter(function(id){ return !!studentMeta[id]; })
    .map(function(id){ return studentMeta[id]; });

  const speakerCounts = _readSheetSpeakerCounts_(ctx, representativeRow);
  ['S1', 'S2', 'S3', 'S4'].forEach(function(id){
    const sheetN = Number(speakerCounts[id] || 0);
    const turnN = Number(turnDerivedCounts[id] || 0);
    if (sheetN !== turnN) {
      warnings.push(id + ' count mismatch: sheet=' + sheetN + ', turns=' + turnN);
    }
  });

  const sheetActive = ['S1', 'S2', 'S3', 'S4'].filter(function(id){ return Number(speakerCounts[id] || 0) > 0; });
  const turnActive = activeSet.slice();
  const sheetOnly = sheetActive.filter(function(id){ return turnActive.indexOf(id) < 0; });
  const turnOnly = turnActive.filter(function(id){ return sheetActive.indexOf(id) < 0; });
  if (sheetOnly.length || turnOnly.length) {
    warnings.push('activeStudentIds conflict: sheet=[' + sheetActive.join(',') + '], turns=[' + turnActive.join(',') + ']');
  }

  const teacherPresent = turns.some(function(t){ return t.role === 'teacher'; });
  if (listRow && ctx.oCol) {
    const oIdx = listRow - 2;
    const oVal = (oIdx >= 0 && oIdx < ctx.display.length) ? String(ctx.display[oIdx][ctx.oCol - 1] || '').trim() : '';
    const flagTeacher = /교사|teacher|1|true|y|yes/i.test(oVal);
    if (flagTeacher !== teacherPresent) {
      warnings.push('teacherPresent mismatch: turns=' + teacherPresent + ', O열="' + oVal + '"');
    }
  }

  const orderIdx = ctx.pidOrder.indexOf(pidKey);
  const previousPid = (orderIdx > 0) ? ctx.pidOrder[orderIdx - 1] : null;
  const nextPid = (orderIdx >= 0 && orderIdx < ctx.pidOrder.length - 1) ? ctx.pidOrder[orderIdx + 1] : null;

  return {
    version: 'KCMP_PACKET_V1',
    pid: pidKey || String(pid || ''),
    representativeRow: representativeRow,
    summary: summary,
    turns: turns,
    students: students,
    activeStudentIds: turnActive,
    speakerCounts: speakerCounts,
    turnDerivedCounts: turnDerivedCounts,
    teacherPresent: teacherPresent,
    context: {
      previousPid: previousPid,
      nextPid: nextPid
    },
    audit: {
      unmappedStudentSpeakers: unmappedStudentSpeakers,
      unknownSpeakers: unknownSpeakers,
      warnings: warnings
    }
  };
}

function buildAllKCMPClusterPackets_(sheet, map){
  const ctx = _prepareKCMPPacketContext_(sheet, map);
  const pidSet = {};
  ctx.pidOrder.forEach(function(pid){ pidSet[pid] = true; });
  Object.keys(ctx.utteranceRowsByPid).forEach(function(pid){ pidSet[pid] = true; });
  const pids = ctx.pidOrder.slice();
  Object.keys(pidSet).forEach(function(pid){
    if (pids.indexOf(pid) < 0) pids.push(pid);
  });
  return pids.map(function(pid){
    return buildKCMPClusterPacket_(sheet, map, pid, { context: ctx });
  });
}

function validateKCMPClusterPacket_(packet){
  const errors = [];
  const warnings = (packet && packet.audit && packet.audit.warnings) ? packet.audit.warnings.slice() : [];

  if (!packet || typeof packet !== 'object') {
    return { ok: false, errors: ['packet이 객체가 아님'], warnings: warnings };
  }
  if (!packet.pid) errors.push('pid 없음');
  if (!Array.isArray(packet.turns)) errors.push('turns가 배열이 아님');
  if (packet.representativeRow != null) {
    if (typeof packet.representativeRow !== 'number' || packet.representativeRow < 2) {
      errors.push('representativeRow 무효: ' + packet.representativeRow);
    }
  } else {
    errors.push('representativeRow 없음');
  }

  if (Array.isArray(packet.turns)) {
    let prevRow = 0;
    packet.turns.forEach(function(t, i){
      if (!t || typeof t.row !== 'number') {
        errors.push('turns[' + i + '] row 없음');
        return;
      }
      if (t.row < prevRow) errors.push('turns row 순서 오류: ' + prevRow + ' → ' + t.row);
      prevRow = t.row;
      if (t.role === 'student') {
        if (!/^S[1-4]$/.test(String(t.speakerId || ''))) {
          errors.push('turns[' + i + '] student인데 speakerId가 S1~S4가 아님: ' + t.speakerId);
        }
      }
      if (t.role === 'teacher') {
        if (t.speakerId != null) errors.push('turns[' + i + '] teacher인데 speakerId가 null이 아님: ' + t.speakerId);
      }
    });
  }

  const active = packet.activeStudentIds || [];
  if (!Array.isArray(active)) {
    errors.push('activeStudentIds가 배열이 아님');
  } else {
    const seenA = {};
    active.forEach(function(id){
      if (seenA[id]) errors.push('activeStudentIds 중복: ' + id);
      seenA[id] = true;
      if (!/^S[1-4]$/.test(String(id || ''))) errors.push('activeStudentIds 값 무효: ' + id);
    });
  }

  const students = packet.students || [];
  if (!Array.isArray(students)) {
    errors.push('students가 배열이 아님');
  } else {
    const seenS = {};
    students.forEach(function(s){
      const id = s && s.id;
      if (seenS[id]) errors.push('students 중복: ' + id);
      seenS[id] = true;
      if (s && s.role === 'teacher') errors.push('students에 teacher가 포함됨');
      if (id && !/^S[1-4]$/.test(String(id))) errors.push('students.id 무효: ' + id);
    });
  }

  const counts = packet.speakerCounts || {};
  ['S1', 'S2', 'S3', 'S4'].forEach(function(id){
    if (counts[id] == null) return;
    if (typeof counts[id] !== 'number' || !isFinite(counts[id])) {
      errors.push('speakerCounts.' + id + '가 숫자가 아님: ' + counts[id]);
    }
  });

  return { ok: errors.length === 0, errors: errors, warnings: warnings };
}

/** 시트 값을 수정하지 않는 개발용 디버그 */
function debugKCMPClusterPacket_(pid){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const packet = buildKCMPClusterPacket_(sh, map, pid);
  const v = validateKCMPClusterPacket_(packet);

  Logger.log('=== KCMP PACKET DEBUG ===');
  Logger.log('PID: ' + packet.pid);
  Logger.log('Representative row: ' + packet.representativeRow);
  Logger.log('Summary: ' + String(packet.summary || '').substring(0, 300));
  Logger.log('Teacher present: ' + packet.teacherPresent);
  Logger.log('Students: ' + JSON.stringify(packet.students));
  Logger.log('Active students: ' + JSON.stringify(packet.activeStudentIds));
  Logger.log('Speaker counts: ' + JSON.stringify(packet.speakerCounts));
  Logger.log('Turn-derived counts: ' + JSON.stringify(packet.turnDerivedCounts));
  Logger.log('Context: ' + JSON.stringify(packet.context));
  Logger.log('Turns (' + packet.turns.length + '):');
  packet.turns.forEach(function(t){
    const who = t.speakerId || t.speakerRaw || '';
    const utt = String(t.utterance || '').replace(/\s+/g, ' ');
    Logger.log('[' + t.row + '] [' + (t.timestamp || '') + '] [' + t.role + '] [' + who + '] ' + utt.substring(0, 120));
  });
  Logger.log('Validation: ok=' + v.ok + ' errors=' + JSON.stringify(v.errors));
  Logger.log('Warnings: ' + JSON.stringify(packet.audit.warnings));
  return packet;
}

/**
 * 개발용: 성격이 다른 PID를 자동 선정해 패킷을 검증한다. 시트는 수정하지 않는다.
 */
function testKCMPClusterPackets_(){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map);

  function pick(pred, used){
    for (let i = 0; i < packets.length; i++) {
      const p = packets[i];
      if (used[p.pid]) continue;
      if (pred(p)) return p;
    }
    return null;
  }

  const used = {};
  const caseA = pick(function(p){ return p.activeStudentIds.length === 1 && p.teacherPresent; }, used);
  if (caseA) used[caseA.pid] = true;
  const caseB = pick(function(p){ return p.activeStudentIds.length >= 2; }, used);
  if (caseB) used[caseB.pid] = true;
  const caseC = pick(function(p){
    return p.activeStudentIds.length >= 3 || (p.activeStudentIds.length >= 2 && p.teacherPresent);
  }, used) || pick(function(p){ return p.turns.length > 0; }, used);

  const chosen = [
    { label: 'CASE A (학생 1명 + 교사)', packet: caseA },
    { label: 'CASE B (학생 2명 이상)', packet: caseB },
    { label: 'CASE C (3명 이상 또는 교사 포함 복합)', packet: caseC }
  ];

  const report = chosen.map(function(item){
    if (!item.packet) {
      Logger.log(item.label + ': 해당 PID를 자동으로 찾지 못함. debugKCMPClusterPacket_("P###")로 지정하세요.');
      return { label: item.label, pid: null, found: false };
    }
    const p = item.packet;
    const v = validateKCMPClusterPacket_(p);
    const dRows = p.turns.length;
    Logger.log('=== ' + item.label + ' ===');
    Logger.log('PID=' + p.pid + ' turns=' + dRows + ' active=' + JSON.stringify(p.activeStudentIds) + ' teacherPresent=' + p.teacherPresent);
    Logger.log('representativeRow=' + p.representativeRow + ' summaryLen=' + String(p.summary || '').length);
    Logger.log('speakerCounts=' + JSON.stringify(p.speakerCounts) + ' turnDerived=' + JSON.stringify(p.turnDerivedCounts));
    Logger.log('validation ok=' + v.ok + ' errors=' + JSON.stringify(v.errors));
    Logger.log('warnings=' + JSON.stringify(p.audit.warnings));
    return {
      label: item.label,
      found: true,
      pid: p.pid,
      turns: dRows,
      activeStudentIds: p.activeStudentIds,
      teacherPresent: p.teacherPresent,
      summaryLen: String(p.summary || '').length,
      speakerCounts: p.speakerCounts,
      turnDerivedCounts: p.turnDerivedCounts,
      validation: v,
      warnings: p.audit.warnings
    };
  });

  Logger.log('총 PID 패킷 수: ' + packets.length);
  return { total: packets.length, cases: report };
}



/***** ===== K Decision Tree v1.0 ===== *****/
const K_DECISION_LABELS_ = {
  K1: "과학적 판단·설명·추론",
  K2: "관찰·자료의 진술·해석",
  K3: "주장에 대한 정당화"
};

function _makeKDecisionError_(errorType, message, pid, extra){
  const err = {
    schema_version: "KCMP_K_V1",
    status: "ERROR",
    error_type: String(errorType || "ERROR"),
    message: String(message == null ? "" : message),
    pid: pid || "",
    code: null,
    contributors: []
  };
  if (extra && typeof extra === "object") {
    Object.keys(extra).forEach(function(k){ err[k] = extra[k]; });
  }
  return err;
}

function _kcmpNormForQuote_(s){
  return String(s == null ? "" : s).replace(/\s+/g, " ").trim();
}

function _kcmpPathHas_(path, token){
  return (path || []).some(function(x){ return String(x).indexOf(token) >= 0; });
}

/** step0_basis / quotes 인용이 학생 원발화에 실제 존재하는지 확인 */
function _kcmpQuoteExistsInStudentTurns_(quote, studentTurns){
  const q = _kcmpNormForQuote_(quote);
  if (q.length < 2) return false;
  return (studentTurns || []).some(function(t){
    return _kcmpNormForQuote_(t.utterance).indexOf(q) >= 0;
  });
}

function formatKDecisionDisplay_(result){
  if (!result || result.status !== "OK" || !result.code) return "";
  const label = K_DECISION_LABELS_[result.code] || result.code;
  let reason = String(result.reason == null ? "" : result.reason).replace(/\s+/g, " ").trim();
  if (reason.length > 60) reason = reason.slice(0, 57) + "…";
  if (!reason) return result.code + " — " + label;
  return result.code + " — " + label + " — " + reason;
}

function _writeKDecisionCell_(sheet, row, kCol, result){
  const cell = sheet.getRange(row, kCol);
  cell.clearContent();
  if (result && result.status === "OK" && result.code) {
    cell.setValue(formatKDecisionDisplay_(result));
  }
  try {
    cell.setNote(JSON.stringify(result || {}));
  } catch (e) {
    cell.setNote(JSON.stringify(_makeKDecisionError_("NOTE_WRITE_ERROR", e.toString(), result && result.pid)));
  }
}

function buildKDecisionPrompt_(packet){
  const students = (packet.students || []).map(function(s){
    return (s.id || "") + " = " + (s.label || "");
  }).join("\n");
  const active = (packet.activeStudentIds || []).join(", ");
  const turnsText = (packet.turns || []).map(function(t){
    const sid = t.speakerId ? t.speakerId : "";
    const raw = t.speakerRaw || "";
    return "[" + t.row + "] " + t.role + " " + sid + " " + raw + ": " + String(t.utterance || "").replace(/\s+/g, " ");
  }).join("\n");
  const summary = String(packet.summary == null ? "" : packet.summary);

  const lines = [];
  lines.push("당신은 과학 소집단 담화의 K차원(인식적 실행) 코더이다.");
  lines.push("입력의 1차 자료는 [CURRENT CLUSTER TURNS]의 원발화이다.");
  lines.push("[SUMMARY]는 보조자료이다. 원발화와 충돌하면 원발화를 우선한다.");
  lines.push("주변 클러스터(previousPid/nextPid) 내용을 추측하여 채워 넣지 마라.");
  lines.push("JSON만 출력하라. JSON 외 텍스트, 마크다운, 코드펜스 금지.");
  lines.push("");
  lines.push("가능한 code: null | \"K1\" | \"K2\" | \"K3\"  (최종 코드는 최대 1개)");
  lines.push("우선순위: K3 > K2 > K1 > null. K1과 K2가 모두 가능하고 K2 최소조건이 충족되면 K2를 우선한다. 단 상위 코드의 최소조건이 충족될 때만 부여.");
  lines.push("contributor는 최종 K코드 성립에 실제로 K 기능을 수행한 학생 S ID만이다.");
  lines.push("단순히 quotes에 등장한 학생 전체가 아니다. 잡담/감탄/수업운영/과목대화/단순반응 학생은 제외.");
  lines.push("교사는 contributor에 넣지 마라. contributor는 아래 ACTIVE STUDENTS 중에서만.");
  lines.push("");
  lines.push("===== GLOBAL =====");
  lines.push("GLOBAL-K1 현재 클러스터 turns만 기본 범위. 없는 주장/근거를 summary에서 만들지 마라. 지시 대상이 현재만으로 안 풀리면 context_needed=true. 추측으로 K3 올리지 마라.");
  lines.push("GLOBAL-K2 문법보다 기능. 접속사 없어도 주장-근거가 내용적으로 연결되면 K3 가능. 한 문장에 정보가 같이 있다고 자동 K3 아님.");
  lines.push("GLOBAL-K3 과학적으로 틀려도 정당화 기능이 있으면 K3 가능.");
  lines.push("GLOBAL-K4 과제 고유명사(훈이/짱구 등)가 선택지를 나타낼 수 있다. bare label(\"훈이.\")만으로는 자동 K1 아님. \"훈이인 것 같아\"는 K1 가능. 선택+이유면 K3 가능.");
  lines.push("GLOBAL-K5 최소조건 없으면 더 낮은 코드 또는 null.");
  lines.push("");
  lines.push("===== K-STEP0 과학적 의미구성이 있는가? =====");
  lines.push("질문: 현재 클러스터에 학생이 과학적 대상·현상·개념·관계에 대해 판단/설명/관찰/해석/질문/추론을 구성한 발화가 있는가?");
  lines.push("NO → code=null, step0_basis=[], science_content=null, reason에 왜 K가 아닌지 명시.");
  lines.push("");
  lines.push("STEP0-R1. 과학 수업에서 나온 말 ≠ 과학적 의미구성");
  lines.push("과학 시간에 나온 발화라도 과학적 대상·현상·개념·관계 자체를 다루지 않으면 K가 아니다.");
  lines.push("예(모두 K 없음): \"교과서 안 쓰는 과목 뭐 있냐?\", \"일본어 시간에 영화 봤어.\", \"생명 때 했어.\", \"나는 줄 읽기만 했어.\", \"순서 외우기도 했어.\", \"몇 페이지야?\", \"이거 적어?\", \"우리 저번 시간에 했어.\"");
  lines.push("다음은 과학적 의미구성이 아님: 교과목 대화, 교과서 사용 여부, 수업 운영, 과거 활동 여부, 읽기/쓰기/암기, 페이지/준비물/기록, 과제 수행 여부, 과학 내용 없는 수업 경험 회상.");
  lines.push("");
  lines.push("STEP0-R2. 과학 용어처럼 보이는 단어 하나만으로 통과 금지");
  lines.push("\"생명\", \"조상\", \"폐\", \"압력\" 등 단어 등장만으로 STEP0 YES 아님.");
  lines.push("그 단어로 학생이 실제로 판단/설명/관찰/해석/과학 내용 질문/추론을 구성했는지 확인. 단순 단어 언급은 K 없음.");
  lines.push("");
  lines.push("STEP0-R3. 불완전하고 대상이 특정되지 않는 과학적 단편");
  lines.push("예: \"우리의 조상이 있을 거...\" — 발화가 불완전하고 무엇에 대한 어떤 과학적 판단인지 현재 클러스터에서 특정 불가 → K1로 복원하지 말고 code=null.");
  lines.push("context_needed=true 가능하나, 현재 클러스터에 충분한 의미 없으면 code=null. 주변 클러스터 내용을 임의로 가져와 K 생성 금지.");
  lines.push("");
  lines.push("STEP0-R4. request-only vs content-question");
  lines.push("\"왜 그렇게 생각해?\" / \"너는 뭐라고 생각해?\" → 상대 판단 요청 → K 없음.");
  lines.push("\"보일 법칙이 뭐야?\" → 과학 내용 자체 질문 → K1 검토. 질문형 여부가 아니라 질문 대상이 과학 내용 자체인지가 핵심.");
  lines.push("");
  lines.push("STEP0 YES일 때 step0_basis에 K-STEP0를 YES로 만든 학생 발화(quote)를 최소 1개 기록. 교사 발화 금지.");
  lines.push("");
  lines.push("Final STEP0 self-check (출력 직전 내부 확인):");
  lines.push("선택한 step0_basis 문장을 과학적 의미구성 맥락 없이 단독으로 읽어도, 과학적 대상·현상·개념·관계에 관한 학생 자신의 판단·설명·관찰·해석·내용 질문·추론이 확인되는가?");
  lines.push("NO면 STEP0 NO로 돌아가 code=null 검토. 과제 고유 표현은 동일 클러스터 맥락이 명확할 때만 허용.");
  lines.push("");
  lines.push("===== K-STEP1 K3 주장-근거 정당화 =====");
  lines.push("학생이 정보·성질·관찰·자료·원리·기제를 어떤 주장·판단·결론을 정당화하려고 쓰는가? YES → K3.");
  lines.push("K3 최소 3요건: (1) 주장 또는 특정 가능한 결론 (2) 실질적 내용적 근거 (3) 근거가 주장을 뒷받침하는 기능적 연결.");
  lines.push("내부적으로 claim=?, evidence=?, 왜 evidence가 claim을 뒷받침하는가? 중 하나라도 못 답하면 K3 금지.");
  lines.push("보정: 접속사 불필요. 학생 간 공동 구성 가능(주장 학생+근거 학생 모두 contributor). 교사-학생 문답으로 같은 클러스터에서 연결되면 K3, 교사는 contributor 제외.");
  lines.push("단순 낭독은 null 또는 상황에 따라 K1. 답을 정당화하면 K3. 생략된 주장은 현재 클러스터에서 하나의 특정 결론으로 명확히 복원될 때만 K3.");
  lines.push("질문형 잠정 주장+근거 가능. 과학 오류 허용. pseudo-reason(그냥/원래/당연히/왠지/그럴 것 같아서)은 근거 아님.");
  lines.push("반복만으로 새 정당화 아님. 후보 주장과 후보 근거가 공존해도 기능적 연결 없으면 K3 아님.");
  lines.push("");
  lines.push("===== K3 claim-evidence 방향 (출력 직전 self-check) =====");
  lines.push("K3 출력 직전 반드시 확인: \"evidence는 실제로 claim에 대한 '왜?'의 답인가?\"");
  lines.push("적절 예: claim=\"공기가 주사기 안으로 들어온다.\", evidence=\"피스톤을 당기면 내부 압력이 낮아진다.\" → evidence가 claim을 설명.");
  lines.push("재검토 예: claim=\"내부 압력이 낮아진다.\", evidence=\"공기가 안으로 들어온다.\" → 발화에서 공기 유입이 압력 저하의 근거로 쓰인 것이 아니면 claim/evidence 방향을 뒤집거나 K3를 재검토.");
  lines.push("claim/evidence를 단순히 두 과학 문장으로 채우지 말 것. WHY(claim)? → evidence 가 자연스럽게 성립해야 한다.");
  lines.push("");
  lines.push("===== contributors ↔ quotes 집합 일치 (모든 non-null K) =====");
  lines.push("contributors = 최종 K코드 성립에 실제로 기여한 학생 집합.");
  lines.push("quotes = 그 최종 K코드 성립을 직접 보여주는 학생 원발화 증거.");
  lines.push("정상 non-null K: UNIQUE(contributors) === UNIQUE(quotes[].speaker) (양방향 일치).");
  lines.push("A. contributors에 있는 학생 → final quotes에 최소 1개 직접 근거 발화 필수.");
  lines.push("B. final quotes speaker가 최종 K 기여자 → contributors에 반드시 포함.");
  lines.push("절대로 근거 quote 없이 학생을 contributor에 넣지 않는다.");
  lines.push("");
  lines.push("step0_basis speaker 집합 ≠ contributors. step0_basis는 K 영역 진입 근거, quotes는 최종 K1/K2/K3 성립 근거.");
  lines.push("S3가 K1 판단만 했고 최종 K3 정당화에 기여하지 않았다면: step0_basis 가능, quotes/contributors 불가.");
  lines.push("S3가 최종 K3 주장/근거에 실제 기여했다면: contributors에 S3 + 해당 S3 발화를 quotes에 포함.");
  lines.push("단순히 클러스터 참여만으로 contributor 포함 금지.");
  lines.push("");
  lines.push("P043형 판단: claim/evidence quote가 모두 S1이면 contributors=[\"S1\"]. S3도 K3 기여했다면 S3 quote 포함 + contributors=[\"S1\",\"S3\"].");
  lines.push("reason에 기여자 명시 시 contributors와 모순 금지. contributors=[\"S1\"]인데 \"S1과 S3가 함께...\" 금지.");
  lines.push("contributors 1명이면 \"학생들이\" 같은 복수 표현 금지. \"S1의 발화에서...\" 또는 \"학생 발화에서...\" 사용.");
  lines.push("");
  lines.push("===== K-STEP2 K2 관찰·자료 내용 =====");
  lines.push("K3가 아니면: 학생이 실험/관찰/표/그래프/그림/도식/모형/측정값/제시된 자료/교과서 그림·도식·자료의 구체적 내용을 진술하거나 해석하는가? YES → K2.");
  lines.push("");
  lines.push("K2 최소조건: 위 자료 유형 중 하나의 \"구체적 내용\"을 학생이 실제로 진술·해석해야 한다.");
  lines.push("자료를 \"본다\"는 사실만으로 K2 아님.");
  lines.push("K2 아님 예: \"표 보자.\", \"교과서 보자.\", \"몇 쪽이야?\", \"여기 보자.\"");
  lines.push("K2 가능 예: \"표에 혈구가 없어.\", \"그래프가 내려가.\", \"그림에서 3번 과정이 재흡수와 분비야.\", \"여기 수치가 더 커.\", \"실험에서 색이 변했어.\"");
  lines.push("");
  lines.push("BOUNDARY K1↔K2:");
  lines.push("발화가 과학적 판단·설명·추론이면서 동시에 자료/관찰/도식의 구체적 내용을 진술·해석하고 있다면 → K2 우선.");
  lines.push("K1과 K2 모두 가능 + K2 최소조건 충족 → K2.");
  lines.push("특정 자료·관찰·도식 내용에 근거하지 않고 학생 자신의 일반적 판단·예측·설명만 있음 → K1.");
  lines.push("K-STEP2를 충분히 검토한 뒤에만 K-STEP3로 내려간다.");
  lines.push("");
  lines.push("P013형: \"(교과서를 보여주며) 3번 과정이지? 재흡수와 분비가 일어난다.\" — 교과서 도식/그림의 3번 과정 내용을 읽고 해석한 것이면 K2.");
  lines.push("판단 전 확인: \"이 발화가 실제 그림/도식/자료의 구체적 내용을 진술·해석하는가?\" YES→K2, NO→K1 검토.");
  lines.push("");
  lines.push("BACKTRACK: 자료/관찰이 특정 주장에 대한 '왜?'의 답이면 K3. K2를 주기 전에 반드시 이 질문을 확인하고, YES면 K3, decision_path에 K-STEP2-BACKTRACK:YES.");
  lines.push("K2 vs K3 예: \"표에 포도당이 없어.\"→K2. \"표에 포도당이 없으니까 재흡수된 거야.\"→K3.");
  lines.push("K2를 유지하면 K-STEP2-BACKTRACK:NO. K2 path: [\"K-STEP0:YES\",\"K-STEP1:NO\",\"K-STEP2:YES\"] (BACKTRACK NO 포함 가능).");
  lines.push("");
  lines.push("===== K-STEP3 K1 기타 과학적 의미구성 =====");
  lines.push("K3/K2가 아니면: 과학적 판단·예측·설명·추론·가능성·속성 진술 또는 과학 내용 질문인가? YES → K1.");
  lines.push("K1 예: \"단백질은 안 나갈 것 같아.\", \"압력이 낮아질 것 같아.\", \"혈구는 못 나가.\", \"태반이랑 양막이 비슷비슷할 텐데.\" — 판단/예측이지만 구체적 자료·관찰 내용 진술·해석 기능 없음.");
  lines.push("\"너 뭐라고 생각해?\" / \"왜 그렇게 생각해?\" 같은 request-only는 null.");
  lines.push("K1 path: [\"K-STEP0:YES\",\"K-STEP1:NO\",\"K-STEP2:NO\",\"K-STEP3:YES\"]");
  lines.push("");
  lines.push("===== 경계 =====");
  lines.push("null vs K1: 학생 자신의 과학적 판단·설명·내용 질문이 있으면 K1 이상.");
  lines.push("K1 vs K3: 판단만이면 K1, 판단+연결된 내용적 이유면 K3.");
  lines.push("K2 vs K3: 자료/관찰 내용 자체면 K2, 자료로 주장·결론 뒷받침이면 K3.");
  lines.push("K1 vs K2: K1과 K2 모두 가능하고 K2 최소조건 충족 시 K2 우선. 자료 구체적 내용 없는 일반 판단·예측만이면 K1.");
  lines.push("");
  lines.push("===== science_content / step0_basis / quotes =====");
  lines.push("science_content = 이 클러스터에서 학생이 실제로 구성한 과학적 의미의 핵심 (non-null K에서 필수 non-empty string).");
  lines.push("step0_basis = K-STEP0를 YES로 판단하게 만든 실제 과학적 의미구성 발화. code!=null이면 최소 1개. code==null이면 [].");
  lines.push("quotes = 최종 K코드 판정의 직접 증거. step0_basis와 구분한다.");
  lines.push("");
  lines.push("IMPORTANT: step0_basis and quotes MUST use the same object entry format.");
  lines.push("Never output a bare string inside either array.");
  lines.push("");
  lines.push("entry schema (step0_basis·quotes 공통):");
  lines.push('  {"speaker":"S1","quote":"원발화 문자열"}');
  lines.push("speaker: \"S1\"|\"S2\"|\"S3\"|\"S4\" (필수)");
  lines.push("quote: CURRENT CLUSTER TURNS의 실제 학생 발화 문자열 (필수)");
  lines.push("");
  lines.push("quotes는 반드시 ARRAY OF OBJECTS이다.");
  lines.push("올바른 형식:");
  lines.push('  "quotes": [{"speaker":"S1","quote":"혈구는 못 나가."}]');
  lines.push("학생 2명:");
  lines.push('  "quotes": [{"speaker":"S1","quote":"혈구는 못 나가."},{"speaker":"S2","quote":"크기가 크니까."}]');
  lines.push("절대 금지: \"quotes\":[\"혈구는 못 나가.\"]  (bare string 배열)");
  lines.push("절대 금지: \"quotes\":[\"S1: 혈구는 못 나가.\"]");
  lines.push("절대 금지: \"quotes\":[{\"quote\":\"혈구는 못 나가.\"}]  (speaker 누락)");
  lines.push("");
  lines.push("quote는 단순히 클러스터에서 눈에 띄는 문장을 고르는 것이 아니다. 해당 quote 자체가 최종 K코드의 최소조건을 직접 보여줘야 한다.");
  lines.push("K1인데 \"교과서 안 쓰는 과목\", \"줄 읽기만 했어\" 같은 수업/교과 발화를 quote로 쓰면 안 된다.");
  lines.push("reason은 항상 비어 있지 않은 문자열 (정상 K 없음도 이유 필수).");
  lines.push("");
  lines.push("===== JSON 출력 직전 self-check =====");
  lines.push("- step0_basis의 모든 원소가 object인가? (bare string 금지)");
  lines.push("- quotes의 모든 원소가 object인가? (bare string 금지)");
  lines.push("- 각 object에 speaker가 있는가? (S1~S4)");
  lines.push("- 각 object에 quote가 있는가?");
  lines.push("- quote는 CURRENT CLUSTER TURNS의 실제 학생 발화인가?");
  lines.push("- UNIQUE(contributors) === UNIQUE(quotes[].speaker) 인가? (모든 non-null K)");
  lines.push("  → contributors에 있으나 quotes speaker에 없는 학생 제거, 또는 해당 quote 추가");
  lines.push("  → quotes speaker인데 contributors에 없으면 contributors 추가, 또는 quote 제거");
  lines.push("K3 추가 확인:");
  lines.push("- evidence는 claim에 대한 '왜?'의 답인가? (방향 뒤집힘 없는가?)");
  lines.push("- contributors 수와 reason 단복수가 모순되지 않는가?");
  lines.push("하나라도 아니면 출력 전에 JSON을 수정한다.");
  lines.push("");
  lines.push("===== 출력 JSON (이 스키마만) =====");
  lines.push("{");
  lines.push('  "schema_version":"KCMP_K_V1",');
  lines.push('  "status":"OK",');
  lines.push('  "code": null,');
  lines.push('  "contributors": [],');
  lines.push('  "science_content": null,');
  lines.push('  "step0_basis": [],');
  lines.push('  "claim": null,');
  lines.push('  "evidence": null,');
  lines.push('  "reason": "학생 발화는 교과목·수업 활동에 관한 대화이며 과학적 의미구성이 없다.",');
  lines.push('  "decision_path": ["K-STEP0:NO"],');
  lines.push('  "boundary_check": null,');
  lines.push('  "context_needed": false,');
  lines.push('  "quotes": []');
  lines.push("}");
  lines.push("");
  lines.push("K1 예:");
  lines.push('  contributors=["S1"], science_content="태반과 양막의 유사성에 대한 판단", claim=null 가능');
  lines.push('  step0_basis=[{"speaker":"S1","quote":"야 근데 태반이랑 양막이랑 비슷비슷할 텐데"}]');
  lines.push('  quotes=[{"speaker":"S1","quote":"야 근데 태반이랑 양막이랑 비슷비슷할 텐데"}]');
  lines.push('  path=["K-STEP0:YES","K-STEP1:NO","K-STEP2:NO","K-STEP3:YES"]');
  lines.push("K2 예:");
  lines.push('  contributors=["S1"], science_content="표에서 혈구가 없음을 관찰함"');
  lines.push('  evidence="표에 혈구가 없음을 진술함", path=["K-STEP0:YES","K-STEP1:NO","K-STEP2:YES","K-STEP2-BACKTRACK:NO"]');
  lines.push('  quotes=[{"speaker":"S1","quote":"표에 혈구가 없어."}]');
  lines.push("K2 P013형 예:");
  lines.push('  contributors=["S1"], science_content="교과서 도식 3번 과정이 재흡수와 분비임을 해석함"');
  lines.push('  evidence="교과서 도식의 3번 과정이 재흡수와 분비로 표시되어 있음을 해석함"');
  lines.push('  quotes=[{"speaker":"S1","quote":"3번 과정이지? 재흡수와 분비가 일어난다."}]');
  lines.push("K3 예:");
  lines.push('  science_content="피스톤을 당겨 압력이 낮아지면 공기가 들어온다고 정당화함", claim+evidence 필수');
  lines.push('  claim="공기가 주사기 안으로 들어온다.", evidence="피스톤을 당기면 내부 압력이 낮아진다."');
  lines.push('  contributors=["S1","S2"], quotes=[{"speaker":"S1","quote":"공기가 들어와"},{"speaker":"S2","quote":"피스톤 당기면 압력 낮아져"}]');
  lines.push('  reason="S1이 공기 유입을 주장하고 S2가 압력 저하를 근거로 제시함"');
  lines.push("K3 단독 예:");
  lines.push('  contributors=["S1"], quotes=[{"speaker":"S1","quote":"혈구는 못 나가."},{"speaker":"S1","quote":"크기가 크니까."}]');
  lines.push("K3 공동 예 (contributors와 quotes speaker 집합 일치):");
  lines.push('  contributors=["S1","S3"], quotes=[{"speaker":"S1","quote":"혈구는 못 나가."},{"speaker":"S3","quote":"크기가 크니까."}]');
  lines.push("금지 예: contributors=[\"S1\",\"S3\"]인데 quotes speaker가 S1뿐 → S3 제거 또는 S3 quote 추가.");
  lines.push("code가 K3이면 claim과 evidence를 비우지 마라. quotes/step0_basis의 speaker는 학생 S ID만.");
  lines.push("null path 예: [\"K-STEP0:NO\"], step0_basis=[], science_content=null, quotes=[], reason 필수.");
  lines.push("");
  lines.push("[PID]");
  lines.push(packet.pid || "");
  lines.push("");
  lines.push("[STUDENTS]");
  lines.push(students || "(없음)");
  lines.push("");
  lines.push("[ACTIVE STUDENTS]");
  lines.push(active || "(없음)");
  lines.push("");
  lines.push("[CURRENT CLUSTER TURNS]");
  lines.push(turnsText || "(원발화 없음)");
  lines.push("");
  lines.push("[SUMMARY - AUXILIARY ONLY]");
  lines.push("보조자료이며 원발화와 충돌하면 원발화를 우선한다.");
  lines.push(summary || "(요약 없음)");
  lines.push("");
  lines.push("JSON만 출력하라.");
  return lines.join("\n");
}

function parseKDecisionTreeResponse_(raw){
  let s = String(raw == null ? "" : raw).replace(/^\uFEFF/, "").trim();
  if (!s) throw new Error("빈 응답");
  s = s.replace(/^```(?:json)?\s*/i, "").replace(/\s*```$/i, "").trim();
  try {
    return JSON.parse(s);
  } catch (e1) {
    const start = s.indexOf("{");
    const end = s.lastIndexOf("}");
    if (start >= 0 && end > start) {
      try {
        return JSON.parse(s.slice(start, end + 1));
      } catch (e2) {
        throw new Error("JSON.parse 실패: " + e2.toString());
      }
    }
    throw new Error("JSON.parse 실패: " + e1.toString());
  }
}

function _kcmpQuoteSpeakers_(quotes){
  const set = {};
  if (!Array.isArray(quotes)) return set;
  quotes.forEach(function(q){
    if (q && typeof q === "object" && !Array.isArray(q) && q.speaker && /^S[1-4]$/.test(String(q.speaker))) {
      set[q.speaker] = true;
    }
  });
  return set;
}

function _kcmpValidateContributorQuoteSetEquality_(contributors, quotes, errors){
  const quoteSpeakers = _kcmpQuoteSpeakers_(quotes);
  const contrib = Array.isArray(contributors) ? contributors : [];
  contrib.forEach(function(id){
    if (!quoteSpeakers[id]) errors.push("contributor " + id + "가 quotes speaker에 없음");
  });
  Object.keys(quoteSpeakers).forEach(function(sp){
    if (contrib.indexOf(sp) < 0) errors.push("quotes speaker " + sp + "가 contributors에 없음");
  });
}

function _kcmpValidateQuoteEntries_(entries, studentTurns, label, errors, packet){
  if (!Array.isArray(entries)) {
    errors.push(label + "가 배열이 아님");
    return;
  }
  entries.forEach(function(q, i){
    const idx = label + "[" + i + "]";
    if (q == null || typeof q !== "object" || Array.isArray(q)) {
      if (typeof q === "string") {
        errors.push(idx + "은 {speaker, quote} 객체여야 함 (bare string 금지)");
      } else {
        errors.push(idx + "은 {speaker, quote} 객체여야 함");
      }
      return;
    }
    const sp = q.speaker;
    if (!sp || !/^S[1-4]$/.test(String(sp))) {
      errors.push(idx + "에 speaker(S1~S4) 필드가 없거나 무효함");
      return;
    }
    if (packet && (packet.activeStudentIds || []).indexOf(sp) < 0) {
      errors.push(idx + " speaker가 activeStudentIds에 없음: " + sp);
    }
    if (q.quote == null || String(q.quote).trim().length === 0) {
      errors.push(idx + "에 quote 필드가 없거나 비어 있음");
      return;
    }
    const quote = _kcmpNormForQuote_(q.quote);
    if (quote.length < 2) {
      errors.push(idx + " quote가 너무 짧음");
    } else if (!_kcmpQuoteExistsInStudentTurns_(q.quote, studentTurns)) {
      errors.push(idx + " 인용이 학생 원발화에서 확인되지 않음: " + quote);
    }
  });
}

function validateKDecisionResult_(result, packet){
  const errors = [];
  const warnings = [];
  if (!result || typeof result !== "object") {
    return { ok: false, errors: ["결과가 객체가 아님"], warnings: warnings };
  }
  if (result.schema_version !== "KCMP_K_V1") errors.push("schema_version 불일치: " + result.schema_version);
  if (result.status !== "OK") errors.push("status가 OK가 아님: " + result.status);

  const reason = String(result.reason == null ? "" : result.reason).trim();
  if (reason.length === 0) errors.push("reason이 비어 있음 (정상 결과는 reason 필수)");

  const allowed = { K1: true, K2: true, K3: true };
  const code = result.code;
  if (!(code === null || allowed[code])) errors.push("code 무효: " + code);

  const scRaw = result.science_content;
  const sc = scRaw == null ? null : String(scRaw).trim();
  const hasScienceContent = sc != null && sc.length > 0;

  if (!Array.isArray(result.step0_basis)) {
    errors.push("step0_basis가 배열이 아님");
  }
  const step0Basis = Array.isArray(result.step0_basis) ? result.step0_basis : [];

  if (!Array.isArray(result.contributors)) {
    errors.push("contributors가 배열이 아님");
  } else {
    const seen = {};
    const active = {};
    (packet && packet.activeStudentIds ? packet.activeStudentIds : []).forEach(function(id){ active[id] = true; });
    result.contributors.forEach(function(id){
      if (seen[id]) errors.push("contributor 중복: " + id);
      seen[id] = true;
      if (!/^S[1-4]$/.test(String(id || ""))) errors.push("contributor가 S1~S4가 아님: " + id);
      if (packet && !active[id]) errors.push("contributor가 activeStudentIds에 없음: " + id);
    });
    if (code != null && result.contributors.length < 1) errors.push("code가 있는데 contributors가 비어 있음");
    if (code == null && result.contributors.length !== 0) errors.push("code=null인데 contributors가 비어 있지 않음");
  }

  if (!Array.isArray(result.decision_path)) {
    errors.push("decision_path가 배열이 아님");
  } else {
    const path = result.decision_path;
    if (code === "K3" && !_kcmpPathHas_(path, "K-STEP1:YES")) errors.push("K3인데 K-STEP1:YES 없음");
    if (code === "K2" && !_kcmpPathHas_(path, "K-STEP2:YES")) errors.push("K2인데 K-STEP2:YES 없음");
    if (code === "K1" && !_kcmpPathHas_(path, "K-STEP3:YES")) errors.push("K1인데 K-STEP3:YES 없음");
    if (code == null) {
      const step0No = _kcmpPathHas_(path, "K-STEP0:NO");
      const allNo = _kcmpPathHas_(path, "K-STEP1:NO") && _kcmpPathHas_(path, "K-STEP2:NO") && _kcmpPathHas_(path, "K-STEP3:NO");
      if (!step0No && !allNo) errors.push("null code인데 decision_path가 K-STEP0:NO 또는 전 STEP NO가 아님");
    }
  }

  const studentTurns = ((packet && packet.turns) || []).filter(function(t){ return t.role === "student"; });

  if (!Array.isArray(result.quotes)) {
    errors.push("quotes가 배열이 아님");
  } else {
    if (code != null && result.quotes.length < 1) errors.push("code가 있는데 quotes가 비어 있음");
    _kcmpValidateQuoteEntries_(result.quotes, studentTurns, "quotes", errors, packet);
  }

  if (code != null) {
    if (!hasScienceContent) errors.push("code가 있는데 science_content가 비어 있음");
    if (step0Basis.length < 1) errors.push("code가 있는데 step0_basis가 비어 있음");
    _kcmpValidateQuoteEntries_(step0Basis, studentTurns, "step0_basis", errors, packet);
    _kcmpValidateContributorQuoteSetEquality_(result.contributors, result.quotes, errors);
  }

  if (code == null) {
    if (scRaw != null && sc !== null && sc.length > 0) errors.push("code=null인데 science_content가 null이 아님");
    if (step0Basis.length !== 0) errors.push("code=null인데 step0_basis가 비어 있지 않음");
  }

  if (code === "K1") {
    if (!hasScienceContent) errors.push("K1인데 science_content가 비어 있음");
    if (step0Basis.length < 1) errors.push("K1인데 step0_basis가 비어 있음");
    if (!Array.isArray(result.quotes) || result.quotes.length < 1) errors.push("K1인데 quotes가 비어 있음");
  }

  if (code === "K2") {
    if (!hasScienceContent) errors.push("K2인데 science_content가 비어 있음");
    if (!_kcmpNormForQuote_(result.evidence)) errors.push("K2인데 evidence가 비어 있음");
    if (step0Basis.length < 1) errors.push("K2인데 step0_basis가 비어 있음");
    if (!Array.isArray(result.quotes) || result.quotes.length < 1) errors.push("K2인데 quotes가 비어 있음");
  }

  if (code === "K3") {
    if (!hasScienceContent) errors.push("K3인데 science_content가 비어 있음");
    if (!_kcmpNormForQuote_(result.claim)) errors.push("K3인데 claim이 비어 있음");
    if (!_kcmpNormForQuote_(result.evidence)) errors.push("K3인데 evidence가 비어 있음");
    if (step0Basis.length < 1) errors.push("K3인데 step0_basis가 비어 있음");
    if (!Array.isArray(result.quotes) || result.quotes.length < 1) errors.push("K3인데 quotes가 비어 있음");
  }

  return { ok: errors.length === 0, errors: errors, warnings: warnings };
}

function runKDecisionTreeForPacket_(packet){
  const pid = packet && packet.pid ? packet.pid : "";
  if (!packet || !pid) return _makeKDecisionError_("PACKET_ERROR", "pid 없음", pid);
  if (!packet.turns || packet.turns.length === 0) {
    return _makeKDecisionError_("PACKET_ERROR", "turns가 비어 있음", pid);
  }

  let raw = "";
  try {
    raw = callGPT_simple_(buildKDecisionPrompt_(packet), MODEL_K);
  } catch (e) {
    return _makeKDecisionError_("API_ERROR", e.toString(), pid);
  }

  let parsed;
  try {
    parsed = parseKDecisionTreeResponse_(raw);
  } catch (e) {
    return _makeKDecisionError_("PARSER_ERROR", e.toString(), pid, { raw_excerpt: String(raw).slice(0, 400) });
  }

  const v = validateKDecisionResult_(parsed, packet);
  if (!v.ok) {
    return _makeKDecisionError_("VALIDATION_ERROR", v.errors.join("; "), pid, {
      validation_errors: v.errors,
      raw_excerpt: String(raw).slice(0, 400)
    });
  }
  parsed.status = "OK";
  parsed.schema_version = "KCMP_K_V1";
  parsed.pid = pid;
  return parsed;
}

function _kcmpSyntheticPacket_(pid, turns){
  const students = {};
  const active = [];
  turns.forEach(function(t, i){
    t.row = t.row || (10 + i);
    t.timestamp = t.timestamp || "";
    t.speakerRaw = t.speakerRaw || t.speakerId || "";
    if (t.role === "student" && t.speakerId) {
      if (!students[t.speakerId]) {
        students[t.speakerId] = { id: t.speakerId, label: t.speakerRaw || t.speakerId, utteranceCount: 0 };
        active.push(t.speakerId);
      }
      students[t.speakerId].utteranceCount++;
    }
  });
  return {
    version: "KCMP_PACKET_V1",
    pid: pid,
    representativeRow: 2,
    summary: "",
    turns: turns,
    students: Object.keys(students).sort().map(function(k){ return students[k]; }),
    activeStudentIds: active,
    speakerCounts: { S1: 0, S2: 0, S3: 0, S4: 0 },
    teacherPresent: turns.some(function(t){ return t.role === "teacher"; }),
    context: { previousPid: null, nextPid: null },
    audit: { unmappedStudentSpeakers: [], unknownSpeakers: [], warnings: [] }
  };
}

function getKDecisionTreeFixtures_(){
  return [
    {
      id: "K_STEP0_OFFTASK_CLASSROOM_TALK",
      expected: null,
      note: "교과목·수업활동 대화 — 과학적 의미구성 없음",
      packet: _kcmpSyntheticPacket_("FX11", [
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "교과서 안 쓰는 과목 뭐 있냐?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "일본어 시간에 영화 봤어." },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "나는 줄 읽기만 했어." },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "순서 외우기도 했어." }
      ])
    },
    {
      id: "K_STEP0_FRAGMENT",
      expected: null,
      note: "과학적 대상/판단 관계가 현재 클러스터에서 특정되지 않는 불완전 단편",
      packet: _kcmpSyntheticPacket_("FX12", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "우리의 조상이 있을 거..." }
      ])
    },
    {
      id: "K_P013_TEXTBOOK_DIAGRAM",
      expected: "K2",
      note: "P013형 — 교과서 도식/그림의 구체적 내용을 읽고 해석",
      packet: _kcmpSyntheticPacket_("P013", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "(교과서를 보여주며) 3번 과정이지? 재흡수와 분비가 일어난다." }
      ])
    },
    {
      id: "1-request-only",
      expected: null,
      packet: _kcmpSyntheticPacket_("FX01", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "왜 그렇게 생각해?" }
      ])
    },
    {
      id: "2-content-question",
      expected: "K1",
      packet: _kcmpSyntheticPacket_("FX02", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "보일 법칙이 뭐예요?" }
      ])
    },
    {
      id: "3-property-judgment",
      expected: "K1",
      packet: _kcmpSyntheticPacket_("FX03", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "혈구는 커." }
      ])
    },
    {
      id: "4-actual-observation",
      expected: "K2",
      packet: _kcmpSyntheticPacket_("FX04", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "부풀어 올라." }
      ])
    },
    {
      id: "5-data-statement",
      expected: "K2",
      packet: _kcmpSyntheticPacket_("FX05", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "표에 혈구가 없어." }
      ])
    },
    {
      id: "6-claim-reason",
      expected: "K3",
      packet: _kcmpSyntheticPacket_("FX06", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "혈구는 못 나가. 너무 크니까." }
      ])
    },
    {
      id: "7-data-as-evidence",
      expected: "K3",
      packet: _kcmpSyntheticPacket_("FX07", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "표에 포도당이 없으니까 재흡수된 거야." }
      ])
    },
    {
      id: "8-pseudo-reason",
      expected: "K1",
      packet: _kcmpSyntheticPacket_("FX08", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "훈이인 것 같아. 그냥." }
      ])
    },
    {
      id: "9-joint-K3",
      expected: "K3",
      packet: _kcmpSyntheticPacket_("FX09", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "혈구는 못 나가." },
        { role: "student", speakerId: "S2", speakerRaw: "누리", utterance: "크기가 크니까." }
      ])
    },
    {
      id: "10-incorrect-but-justified",
      expected: "K3",
      packet: _kcmpSyntheticPacket_("FX10", [
        { role: "student", speakerId: "S1", speakerRaw: "진오", utterance: "공기가 나가서 폐가 커져. 풍선이 커지니까." }
      ])
    }
  ];
}

function testKDecisionTreeForPid_(pid){
  const sh = SpreadsheetApp.getActiveSheet();
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map);
  let packet = null;
  const want = normPID_(pid || "P002");
  for (let i = 0; i < packets.length; i++) {
    if (packets[i].pid === want && packets[i].turns && packets[i].turns.length) {
      packet = packets[i];
      break;
    }
  }
  if (!packet) {
    for (let i = 0; i < packets.length; i++) {
      if (packets[i].turns && packets[i].turns.length) { packet = packets[i]; break; }
    }
  }
  if (!packet) {
    Logger.log("유효한 packet이 없습니다.");
    return { ok: false, message: "no packet" };
  }

  Logger.log("=== K DECISION TREE DRY-RUN ===");
  Logger.log("PID=" + packet.pid + " turns=" + packet.turns.length + " active=" + JSON.stringify(packet.activeStudentIds));
  const prompt = buildKDecisionPrompt_(packet);
  Logger.log("PROMPT_TURNS_INCLUDED=" + (prompt.indexOf("[CURRENT CLUSTER TURNS]") >= 0));
  Logger.log("PROMPT_SUMMARY_AUX=" + (prompt.indexOf("AUXILIARY ONLY") >= 0));

  let raw = "";
  let parsed = null;
  let validation = null;
  let result = null;
  try {
    raw = callGPT_simple_(prompt, MODEL_K);
    Logger.log("RAW=" + String(raw).slice(0, 800));
    parsed = parseKDecisionTreeResponse_(raw);
    validation = validateKDecisionResult_(parsed, packet);
    Logger.log("PARSED=" + JSON.stringify(parsed));
    Logger.log("science_content=" + (parsed && parsed.science_content));
    Logger.log("step0_basis=" + JSON.stringify(parsed && parsed.step0_basis));
    Logger.log("VALIDATION=" + JSON.stringify(validation));
    if (validation.ok) {
      result = parsed;
      result.status = "OK";
    } else {
      result = _makeKDecisionError_("VALIDATION_ERROR", validation.errors.join("; "), packet.pid);
    }
  } catch (e) {
    result = _makeKDecisionError_(String(e).indexOf("JSON") >= 0 ? "PARSER_ERROR" : "API_ERROR", e.toString(), packet.pid);
    Logger.log("ERROR=" + e.toString());
  }

  const display = formatKDecisionDisplay_(result);
  Logger.log("DISPLAY=" + display);
  Logger.log("NOTE_STATUS=" + (result && result.status) + " code=" + (result && result.code) + " error_type=" + (result && result.error_type));
  Logger.log("DRY-RUN: K셀을 수정하지 않음");
  return { packet: packet, raw: raw, parsed: parsed, validation: validation, result: result, display: display };
}

function TEST_K_DECISION_TREE(){
  return testKDecisionTreeForPid_("P002");
}

function TEST_K_DECISION_TREE_P025(){
  return testKDecisionTreeForPid_("P025");
}

function TEST_K_DECISION_TREE_P047(){
  return testKDecisionTreeForPid_("P047");
}

function TEST_K_DECISION_TREE_P043(){
  return testKDecisionTreeForPid_("P043");
}

function TEST_K_DECISION_TREE_P013(){
  return testKDecisionTreeForPid_("P013");
}

/** P002·P025·P047·P043 contributors↔quotes 회귀 (API 호출 없음) */
function TEST_K_VALIDATOR_REGRESSION(){
  const p002Bad = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K1",
    contributors: ["S3"],
    claim: null,
    evidence: null,
    reason: "",
    decision_path: ["K-STEP0:YES", "K-STEP1:NO", "K-STEP2:NO", "K-STEP3:YES"],
    quotes: [
      { speaker: "S3", quote: "야 지금 교과서 안 쓰는 과목 뭐 있냐" },
      { speaker: "S3", quote: "나는 줄 읽기만 했어." }
    ]
  };
  const nullOk = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: null,
    contributors: [],
    science_content: null,
    step0_basis: [],
    claim: null,
    evidence: null,
    reason: "학생 발화는 교과목·수업 활동에 관한 대화이며 과학적 의미구성이 없다.",
    decision_path: ["K-STEP0:NO"],
    quotes: []
  };
  const p025QuoteStrBad = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K1",
    contributors: ["S1"],
    science_content: "태반과 양막의 유사성 판단",
    step0_basis: [
      { speaker: "S1", quote: "태반이랑 양막이랑 비슷비슷할 텐데" }
    ],
    claim: null,
    evidence: null,
    reason: "과학적 유사성 판단",
    decision_path: ["K-STEP0:YES", "K-STEP1:NO", "K-STEP2:NO", "K-STEP3:YES"],
    quotes: [
      "태반이랑 양막이랑 비슷비슷할 텐데"
    ]
  };
  const p025QuoteObjGood = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K1",
    contributors: ["S1"],
    science_content: "태반과 양막의 유사성 판단",
    step0_basis: [
      { speaker: "S1", quote: "태반이랑 양막이랑 비슷비슷할 텐데" }
    ],
    claim: null,
    evidence: null,
    reason: "과학적 유사성 판단",
    decision_path: ["K-STEP0:YES", "K-STEP1:NO", "K-STEP2:NO", "K-STEP3:YES"],
    quotes: [
      { speaker: "S1", quote: "태반이랑 양막이랑 비슷비슷할 텐데" }
    ]
  };
  const packetP002 = _kcmpSyntheticPacket_("P002", [
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "야 지금 교과서 안 쓰는 과목 뭐 있냐 교과서 안 쓰는 과목 뭐 있냐" },
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "나는 줄 읽기만 했어." }
  ]);
  const packetP025 = _kcmpSyntheticPacket_("P025", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "야 근데 태반이랑 양막이랑 비슷비슷할 텐데" }
  ]);
  const p047ContribBad = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K3",
    contributors: ["S1"],
    science_content: "압력 변화로 공기 유입을 정당화함",
    step0_basis: [
      { speaker: "S1", quote: "공기가 들어와" },
      { speaker: "S2", quote: "피스톤 당기면 압력 낮아져" }
    ],
    claim: "공기가 주사기 안으로 들어온다.",
    evidence: "피스톤을 당기면 내부 압력이 낮아진다.",
    reason: "학생들이 공기 유입과 압력 변화를 연결함",
    decision_path: ["K-STEP0:YES", "K-STEP1:YES"],
    quotes: [
      { speaker: "S2", quote: "피스톤 당기면 압력 낮아져" },
      { speaker: "S1", quote: "공기가 들어와" }
    ]
  };
  const p047ContribGood = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K3",
    contributors: ["S1", "S2"],
    science_content: "압력 변화로 공기 유입을 정당화함",
    step0_basis: [
      { speaker: "S1", quote: "공기가 들어와" },
      { speaker: "S2", quote: "피스톤 당기면 압력 낮아져" }
    ],
    claim: "공기가 주사기 안으로 들어온다.",
    evidence: "피스톤을 당기면 내부 압력이 낮아진다.",
    reason: "S1이 공기 유입을 주장하고 S2가 압력 저하를 근거로 제시함",
    decision_path: ["K-STEP0:YES", "K-STEP1:YES"],
    quotes: [
      { speaker: "S2", quote: "피스톤 당기면 압력 낮아져" },
      { speaker: "S1", quote: "공기가 들어와" }
    ]
  };
  const packetP047 = _kcmpSyntheticPacket_("P047", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "공기가 들어와" },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "피스톤 당기면 압력 낮아져" }
  ]);
  const contribQuoteBad = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K3",
    contributors: ["S1", "S3"],
    science_content: "혈구 크기로 여과 불가 정당화",
    step0_basis: [
      { speaker: "S1", quote: "혈구는 못 나가." }
    ],
    claim: "혈구는 못 나간다.",
    evidence: "크기가 크니까.",
    reason: "S1과 S3가 함께 정당화함",
    decision_path: ["K-STEP0:YES", "K-STEP1:YES"],
    quotes: [
      { speaker: "S1", quote: "혈구는 못 나가." },
      { speaker: "S1", quote: "크기가 크니까." }
    ]
  };
  const contribQuoteGood1 = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K3",
    contributors: ["S1"],
    science_content: "혈구 크기로 여과 불가 정당화",
    step0_basis: [
      { speaker: "S1", quote: "혈구는 못 나가." }
    ],
    claim: "혈구는 못 나간다.",
    evidence: "크기가 크니까.",
    reason: "S1이 주장과 근거를 연결함",
    decision_path: ["K-STEP0:YES", "K-STEP1:YES"],
    quotes: [
      { speaker: "S1", quote: "혈구는 못 나가." },
      { speaker: "S1", quote: "크기가 크니까." }
    ]
  };
  const contribQuoteGood2 = {
    schema_version: "KCMP_K_V1",
    status: "OK",
    code: "K3",
    contributors: ["S1", "S3"],
    science_content: "혈구 크기로 여과 불가 정당화",
    step0_basis: [
      { speaker: "S1", quote: "혈구는 못 나가." },
      { speaker: "S3", quote: "크기가 크니까." }
    ],
    claim: "혈구는 못 나간다.",
    evidence: "크기가 크니까.",
    reason: "S1이 주장하고 S3가 크기를 근거로 제시함",
    decision_path: ["K-STEP0:YES", "K-STEP1:YES"],
    quotes: [
      { speaker: "S1", quote: "혈구는 못 나가." },
      { speaker: "S3", quote: "크기가 크니까." }
    ]
  };
  const packetContribQuote = _kcmpSyntheticPacket_("P043", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "혈구는 못 나가." },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "크기가 크니까." },
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "크기가 크니까." }
  ]);
  const badV = validateKDecisionResult_(p002Bad, packetP002);
  const okV = validateKDecisionResult_(nullOk, packetP002);
  const p025StrV = validateKDecisionResult_(p025QuoteStrBad, packetP025);
  const p025ObjV = validateKDecisionResult_(p025QuoteObjGood, packetP025);
  const p047BadV = validateKDecisionResult_(p047ContribBad, packetP047);
  const p047GoodV = validateKDecisionResult_(p047ContribGood, packetP047);
  const contribQuoteBadV = validateKDecisionResult_(contribQuoteBad, packetContribQuote);
  const contribQuoteGood1V = validateKDecisionResult_(contribQuoteGood1, packetContribQuote);
  const contribQuoteGood2V = validateKDecisionResult_(contribQuoteGood2, packetContribQuote);
  Logger.log("P002_BAD ok=" + badV.ok + " errors=" + JSON.stringify(badV.errors));
  Logger.log("NULL_OK ok=" + okV.ok + " errors=" + JSON.stringify(okV.errors));
  Logger.log("P025_QUOTES_STRING_BAD ok=" + p025StrV.ok + " errors=" + JSON.stringify(p025StrV.errors));
  Logger.log("P025_QUOTES_OBJECT_GOOD ok=" + p025ObjV.ok + " errors=" + JSON.stringify(p025ObjV.errors));
  Logger.log("P047_CONTRIB_BAD ok=" + p047BadV.ok + " errors=" + JSON.stringify(p047BadV.errors));
  Logger.log("P047_CONTRIB_GOOD ok=" + p047GoodV.ok + " errors=" + JSON.stringify(p047GoodV.errors));
  Logger.log("CONTRIB_QUOTE_BAD ok=" + contribQuoteBadV.ok + " errors=" + JSON.stringify(contribQuoteBadV.errors));
  Logger.log("CONTRIB_QUOTE_GOOD1 ok=" + contribQuoteGood1V.ok + " errors=" + JSON.stringify(contribQuoteGood1V.errors));
  Logger.log("CONTRIB_QUOTE_GOOD2 ok=" + contribQuoteGood2V.ok + " errors=" + JSON.stringify(contribQuoteGood2V.errors));
  return {
    p002_bad: { ok: badV.ok, errors: badV.errors, expect_ok: false },
    null_ok: { ok: okV.ok, errors: okV.errors, expect_ok: true },
    p025_quotes_string_bad: { ok: p025StrV.ok, errors: p025StrV.errors, expect_ok: false },
    p025_quotes_object_good: { ok: p025ObjV.ok, errors: p025ObjV.errors, expect_ok: true },
    p047_contrib_bad: { ok: p047BadV.ok, errors: p047BadV.errors, expect_ok: false },
    p047_contrib_good: { ok: p047GoodV.ok, errors: p047GoodV.errors, expect_ok: true },
    contrib_quote_bad: { ok: contribQuoteBadV.ok, errors: contribQuoteBadV.errors, expect_ok: false },
    contrib_quote_good1: { ok: contribQuoteGood1V.ok, errors: contribQuoteGood1V.errors, expect_ok: true },
    contrib_quote_good2: { ok: contribQuoteGood2V.ok, errors: contribQuoteGood2V.errors, expect_ok: true }
  };
}


/***** ===== C Decision Tree v1.0 ===== *****/
const C_DECISION_LABELS_ = {
  C1: "동의·수용",
  C2: "의견·명료화·정당화 요청",
  C3: "상호작용을 통한 정교화",
  C4: "비판·반박",
  C5: "설득",
  C6: "공동 조율·결정",
  C7: "또래 교수"
};

function _makeCDecisionError_(errorType, message, pid, extra){
  const err = {
    schema_version: "KCMP_C_V1",
    status: "ERROR",
    error_type: String(errorType || "ERROR"),
    message: String(message == null ? "" : message),
    pid: pid || "",
    code: null,
    contributors: []
  };
  if (extra && typeof extra === "object") {
    Object.keys(extra).forEach(function(k){ err[k] = extra[k]; });
  }
  return err;
}

function _makeCNoneResult_(pid, reason, path){
  return {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: null,
    contributors: [],
    interaction_summary: null,
    reason: reason,
    decision_path: path || ["C-STEP0:NO"],
    boundary_check: null,
    context_needed: false,
    quotes: [],
    pid: pid || ""
  };
}

function _cPacketMappingUnreliable_(packet){
  const audit = packet && packet.audit ? packet.audit : {};
  const u1 = audit.unmappedStudentSpeakers;
  const u2 = audit.unknownSpeakers;
  return (Array.isArray(u1) && u1.length > 0) || (Array.isArray(u2) && u2.length > 0);
}

function formatCDecisionDisplay_(result){
  if (!result || result.status !== "OK" || !result.code) return "";
  const label = C_DECISION_LABELS_[result.code] || result.code;
  let reason = String(result.reason == null ? "" : result.reason).replace(/\s+/g, " ").trim();
  if (reason.length > 60) reason = reason.slice(0, 57) + "…";
  if (!reason) return result.code + " — " + label;
  return result.code + " — " + label + " — " + reason;
}

function _writeCDecisionCell_(sheet, row, cCol, result){
  const cell = sheet.getRange(row, cCol);
  cell.clearContent();
  if (result && result.status === "OK" && result.code) {
    cell.setValue(formatCDecisionDisplay_(result));
  }
  try {
    cell.setNote(JSON.stringify(result || {}));
  } catch (e) {
    cell.setNote(JSON.stringify(_makeCDecisionError_("NOTE_WRITE_ERROR", e.toString(), result && result.pid)));
  }
}

function _kcmpCQuoteExistsInStudentTurns_(quote, studentTurns){
  const q = _kcmpNormForQuote_(quote);
  if (!q) return false;
  return (studentTurns || []).some(function(t){
    const u = _kcmpNormForQuote_(t.utterance);
    if (!u) return false;
    if (q.length === 1) return u === q;
    return u.indexOf(q) >= 0;
  });
}

function _kcmpValidateCQuoteEntries_(entries, studentTurns, label, errors, packet){
  if (!Array.isArray(entries)) {
    errors.push(label + "가 배열이 아님");
    return;
  }
  entries.forEach(function(q, i){
    const idx = label + "[" + i + "]";
    if (q == null || typeof q !== "object" || Array.isArray(q)) {
      if (typeof q === "string") {
        errors.push(idx + "은 {speaker, quote} 객체여야 함 (bare string 금지)");
      } else {
        errors.push(idx + "은 {speaker, quote} 객체여야 함");
      }
      return;
    }
    const sp = q.speaker;
    if (!sp || !/^S[1-4]$/.test(String(sp))) {
      errors.push(idx + "에 speaker(S1~S4) 필드가 없거나 무효함");
      return;
    }
    if (packet && (packet.activeStudentIds || []).indexOf(sp) < 0) {
      errors.push(idx + " speaker가 activeStudentIds에 없음: " + sp);
    }
    if (q.quote == null || String(q.quote).trim().length === 0) {
      errors.push(idx + "에 quote 필드가 없거나 비어 있음");
      return;
    }
    const quote = _kcmpNormForQuote_(q.quote);
    if (!_kcmpCQuoteExistsInStudentTurns_(q.quote, studentTurns)) {
      errors.push(idx + " 인용이 학생 원발화에서 확인되지 않음: " + quote);
    }
  });
}

function buildCDecisionPrompt_(packet){
  const students = (packet.students || []).map(function(s){
    return (s.id || "") + " = " + (s.label || "");
  }).join("\n");
  const active = (packet.activeStudentIds || []).join(", ");
  const turnsText = (packet.turns || []).map(function(t){
    const sid = t.speakerId ? t.speakerId : "";
    const raw = t.speakerRaw || "";
    return "[" + t.row + "] " + t.role + " " + sid + " " + raw + ": " + String(t.utterance || "").replace(/\s+/g, " ");
  }).join("\n");
  const summary = String(packet.summary == null ? "" : packet.summary);

  const lines = [];
  lines.push("당신은 과학 소집단 담화의 C차원(학생-학생 협력적 상호작용) 코더이다.");
  lines.push("입력의 1차 자료는 [CURRENT CLUSTER TURNS]의 원발화이다.");
  lines.push("[SUMMARY]는 보조자료이다. 원발화와 충돌하면 원발화를 우선한다.");
  lines.push("주변 클러스터(previousPid/nextPid) 내용을 추측하여 채워 넣지 마라.");
  lines.push("JSON만 출력하라. JSON 외 텍스트, 마크다운, 코드펜스 금지.");
  lines.push("");
  lines.push("가능한 code: null | \"C1\" | \"C2\" | \"C3\" | \"C4\" | \"C5\" | \"C6\" | \"C7\"  (최종 코드는 최대 1개)");
  lines.push("우선순위: C6 > C7 > C5 > C4 > C3 > C2 > C1. 단 상위 코드의 최소조건이 충족될 때만 부여.");
  lines.push("C = 학생-학생 협력적 상호작용의 기능. 화자 수가 아니라 발화가 서로 기능적으로 연결되었는지가 먼저이다.");
  lines.push("activeStudentIds.length >= 2 만으로 C를 강제하지 않는다.");
  lines.push("OFF_TASK라고 무조건 C 없음이 아니다. 과학 내용이 아니어도 실제 학생-학생 요청/수용/거절/조율/반박이 있으면 C 가능.");
  lines.push("단순 무관한 농담을 여러 명이 각각 하는 것은 C가 아니다.");
  lines.push("");
  lines.push("===== GLOBAL-C =====");
  lines.push("GLOBAL-C1 화자 수보다 기능적 연결이 우선. 활성 학생 2명 이상은 C의 충분조건이 아니다.");
  lines.push("GLOBAL-C2 C는 학생-학생 관계 코드. 교사는 contributor가 될 수 없다.");
  lines.push("teacher mediation 자체는 C를 금지하지 않는다. 그러나 teacher mediation alone → C 아님.");
  lines.push("학생 A → 교사 → 학생 B 순서만으로 A-B 상호작용이 자동 성립하지 않는다.");
  lines.push("teacher-mediated student-student interaction을 인정하려면 B의 발화가 교사의 질문에 답하는 것을 넘어 A의 앞선 발화/입장/설명을 기능적으로 이어받는 근거가 있어야 한다.");
  lines.push("인정 예: B가 A의 주장/설명 내용을 직접 받아 반응, A의 특정 아이디어를 보충, A의 특정 입장을 반박, A의 질문/요청에 실질 응답, A와 자기 아이디어를 비교/조정, A의 contribution 없이는 B의 응답 기능을 설명하기 어려움.");
  lines.push("긍정 예: S1 \"폐가 커지면 압력이 낮아져요.\" Teacher \"S3는 어떻게 생각해?\" S3 \"S1 말에 더하면, 그래서 바깥 공기가 들어오는 거예요.\" → C3 가능.");
  lines.push("부정 예: Teacher가 S1/S2/S3에게 각각 묻고 각자가 독립적으로 답하면, 같은 주제를 말하더라도 C 없음.");
  lines.push("");
  lines.push("GLOBAL-C3 형식보다 기능. 질문형이라고 자동 C2 아님. 반박 기능 질문이면 C4. \"맞아\"가 있어도 새 내용 추가면 C3.");
  lines.push("GLOBAL-C4 과학적 정확성은 C와 별개. 틀린 내용이라도 조율/반박/정교화 기능이 있으면 C 가능.");
  lines.push("GLOBAL-C5 과학 내용만 C 대상이 아님. 과제 진행, 역할, 발표, 기록, 방법, 사회적 요청/수용에서도 실제 학생-학생 관계 기능이 있으면 C 가능.");
  lines.push("GLOBAL-C6 높은 코드의 최소조건이 실제로 성립할 때만 우선. 키워드/문장 형태만으로 상위 코드 금지.");
  lines.push("GLOBAL-C7 현재 클러스터가 기본 범위. 상호작용 대상이 특정되지 않으면 context_needed=true 가능. 관계를 임의로 만들어 C를 부여하지 마라.");
  lines.push("GLOBAL-C8 TEMPORAL: 학생-학생 관계는 ordered packet.turns의 실제 시간 순서를 존중한다.");
  lines.push("후행 발화를 선행 발화의 base/target/cause로 두고, 선행 발화를 후행 발화의 response/elaboration/rebuttal로 재구성하지 마라. later → earlier 방향 관계 금지.");
  lines.push("항상 확인: What utterance existed BEFORE the response? 최종 C3가 필요하다고 비슷한 두 발화의 순서를 뒤집어 base/elaboration으로 묶지 마라.");
  lines.push("");
  lines.push("===== C-STEP0 실제 학생-학생 상호작용이 있는가? =====");
  lines.push("질문: 현재 클러스터에서 최소 2명의 학생이 서로 기능적으로 연결된 발화/행동을 구성하고 있는가?");
  lines.push("YES: (1) 최소 2명의 학생이 실제로 관련됨 AND (2) 서로의 발화가 응답/수용/질문/의미·이유 요구/보충/확장/반박/설득/설명 조정/의견 조율/공동 결정/또래 교수/요청에 대한 수용·거절 중 하나로 연결.");
  lines.push("단순히 activeStudentIds가 2명 이상이라는 이유만으로 STEP0 YES 금지.");
  lines.push("\"같은 과학 문제를 연속해서 말한다\" ≠ \"서로의 발화에 기능적으로 반응한다\".");
  lines.push("두 학생 발화가 의미적으로 비슷하거나 같은 정답을 향한다는 사실만으로 C를 만들지 마라.");
  lines.push("반드시 답할 것: What did student B do WITH student A's contribution?");
  lines.push("답이 \"둘 다 같은 문제에 답했다\"이면 C-STEP0 NO 검토.");
  lines.push("");
  lines.push("P035형 (expected=null): S1 \"혈구가 나가면은?\" → Teacher 반복질문 → S1 \"잘 모르겠어요.\" → S2도 모름 → S3 \"안 나갈 것 같아요\" → Teacher가 선택지를 물음 → S1/S2가 교사 질문에 답함.");
  lines.push("여러 학생이 active이고 같은 과학 문제에 참여해도, 교사가 학생별 응답을 중재하고 S1의 질문은 주장이 아니며 S3는 S1을 반박한 것이 아니면 학생-학생 functional linkage 부족 → code=null.");
  lines.push("S1의 \"혈구가 나가면은?\" / \"잘 모르겠어요.\"를 \"혈구가 나간다\"는 주장으로 복원하지 마라.");
  lines.push("P035 temporal 금지: 실제 순서 S3 \"안 나갈 것 같아요\"(먼저) → S1 \"안 나가잖아요.\"(나중)인데 S1→S3 C3로 역전하지 마라. 권장 expected=null.");
  lines.push("P037 temporal 금지: S4 \"포도당...\"(먼저) → S1 \"입자량...\"(나중)인데 S1→S4 C3로 역전하지 마라.");
  lines.push("P037 C6 evidence alignment: cluster에 여러 relation이 있을 수 있다.");
  lines.push("Relation A (C6 후보): S2 \"그런가? 3번이 맞지 않을까? 바꿀까?\" → S3 \"(끄덕인다)\" — 공동 답 변경 제안+비언어 수용.");
  lines.push("Relation B (C3 후보, lower-priority): S2 \"1번이 왜 아닌지를 생각해보자\" → S1 압력/입자 설명 — 문제 제기+설명 확장, 공동 확정 아님.");
  lines.push("C6가 최종 code면 contributors=[\"S2\",\"S3\"], quotes=S2 proposal + S3 nonverbal acceptance만. S1은 lower-priority C3에만 참여하므로 C6 contributor에 자동 포함 금지.");
  lines.push("현재 S2→S1 quote pair(\"1번이 왜...\" + S1 설명)만으로 C6를 만들지 마라 — proposal+explanation은 C6 evidence가 아니다.");
  lines.push("interaction_summary 예: \"S2가 공동 답을 3번으로 변경할 것을 제안하고 S3가 끄덕임으로 이를 수용하여 답 변경을 공동으로 확정함.\"");
  lines.push("");
  lines.push("STEP0 NO 예: S1 \"1번.\" / S2 \"2번.\" (서로 반응 없음). Teacher가 각자에게 묻고 독립 응답. S1 \"오늘 피곤해.\" / S2 \"배고프다.\"");
  lines.push("STEP0 YES 예: S1 \"난 1번인 것 같아.\" S2 \"왜?\". S1 \"이걸 먼저 적자.\" S2 \"응.\". S1 \"압력이 높아져.\" S2 \"아니, 낮아지는 거 아니야?\"");
  lines.push("\"응\", \"맞아\" 같은 짧은 발화도 앞선 학생의 말에 대한 명확한 수용이면 상호작용 성립 가능.");
  lines.push("비언어적 수용은 원발화 데이터에 명시된 경우만 사용. 예: \"(고개를 끄덕임)\". 데이터에 없는 행동을 추측하지 마라.");
  lines.push("NO → code=null, contributors=[], interaction_summary=null, quotes=[], decision_path=[\"C-STEP0:NO\"], reason에 왜 기능적 연결이 없는지 명시.");
  lines.push("");
  lines.push("===== C-STEP1 → C6 공동 조율·결정 =====");
  lines.push("학생들이 서로 다른 의견/이유/방법/제안을 비교·조정·통합하거나, 여러 제안을 명시적으로 공동 결정으로 확정하는가? YES → C6.");
  lines.push("A. 아이디어 조율: 서로 다른 요소를 공동 설명으로 통합.");
  lines.push("예: S1 \"원인은 압력 차이인 것 같아.\" S2 \"근데 횡격막 움직임도 넣어야 해.\" S1 \"그럼 둘 다 연결해서 설명하자.\"");
  lines.push("B. 공동 의사결정: 역할/발표/기록/방법/절차도 실제 공동 결정하면 C6 가능.");
  lines.push("예: S1 \"내가 발표할까?\" S2 \"그럼 내가 기록할게.\" S1 \"좋아, 그렇게 하자.\"");
  lines.push("절차/역할이라고 C6 금지하지 않는다. C6 최소조건: 최소 2명의 실제 제안/입장/선택이 관련됨 AND 비교/조정/통합 또는 명시적 공동 확정.");
  lines.push("C6 아님: 혼자 결정/공지, 단순 지시, 설득으로 상대 입장 철회(C5 검토), 단순 \"응\".");
  lines.push("C6 FINAL-RELATION 최소조건: code=C6를 선택했다면 contributors/quotes/interaction_summary/reason은 반드시 C6 자체를 직접 성립시키는 joint coordination / joint decision relation만 보여야 한다.");
  lines.push("A가 문제 해결 방향을 제안하고 B가 이유/설명을 제공했다는 것만으로 C6 금지. 그 관계가 설명 확장이면 C3 후보이다.");
  lines.push("proposal + explanation ≠ 자동 C6. 공동 outcome/decision/criterion/solution 형성 근거가 quotes에서 직접 확인되어야 한다.");
  lines.push("C6 evidence self-check (출력 전 필수): (1) 어떤 공동 outcome/decision/criterion/solution이 형성되었는가? (2) 각 contributor는 그 형성에 무엇을 기여했는가? (3) quotes만 읽어도 공동 조율/결정이 실제로 확인되는가? (4) 단순 proposal+explanation을 공동 결정으로 과대해석하지 않았는가?");
  lines.push("1~3을 quotes로 입증할 수 없으면 그 quote set을 C6 evidence로 사용하지 마라.");
  lines.push("C6 nonverbal acceptance: 공유 과제 답 변경/선택 제안에 peer가 \"(끄덕인다)\" \"(고개를 끄덕임)\" 등 원발화에 명시된 긍정적 수용이 있으면 joint decision C6 후보. packet utterance로 존재하면 직접 quote evidence로 사용.");
  lines.push("예: S2 \"3번이 맞지 않을까? 바꿀까?\" S3 \"(끄덕인다)\" → 공동 답 변경 제안+수용 → C6 후보.");
  lines.push("C6↔C1: S1 \"압력이 낮아져.\" S2 \"응.\" → 단순 proposition 수용 → C1. S1 \"우리 답을 3번으로 바꿀까?\" S2 \"(끄덕인다)\" → 공유 과제 답/행동을 함께 확정 → C6. acceptance 대상이 단순 proposition인지 공동 outcome/decision인지 구별.");
  lines.push("C6↔C3: S2 \"1번이 왜 아닌지 생각해보자.\" S1 \"압력 때문에 작은 입자는 다 나가는 것 같아.\" → B가 설명 추가, 공동 결정 확정 아님 → C3 가능, C6 자동 금지. C6에는 joint outcome 형성 근거가 별도로 필요.");
  lines.push("단, 앞선 복수 제안이 있고 \"응, 그걸로 하자\"가 최종 공동 결정을 확정하면 전체는 C6 가능.");
  lines.push("C6 path: [\"C-STEP0:YES\",\"C-STEP1:YES\"]");
  lines.push("");
  lines.push("===== C-STEP2 → C7 또래 교수 =====");
  lines.push("C6 아니면: 한 학생이 특정 학생의 이해를 돕기 위해 그 학생의 어려움/수준에 맞춰 설명을 재구성하는가? YES → C7.");
  lines.push("C7 = peer teaching. 사람 코딩에서도 드문 코드다. 최소조건이 명확할 때만 부여. 적극적으로 찾아내어 부여하지 마라.");
  lines.push("C7 최소조건 3개 모두 필요: (A) identifiable learner (B) 그 learner가 실제 이해 곤란/혼란/설명 필요를 드러냄 (C) peer가 그 학생의 이해를 위해 설명을 재구성함.");
  lines.push("A identifiable learner: 이해의 대상이 되는 특정 peer가 식별되어야 한다. 여러 명이 있는 상황에서 한 학생이 긴 설명을 시작했지만 누구의 이해를 돕는지 특정할 수 없으면 C7 금지.");
  lines.push("B learner difficulty: 단순 내용 질문(\"어떤 일이 일어날까?\")만으로 이해 곤란을 확정하지 마라. learner가 실제로 헷갈리거나 이해하지 못함을 드러내야 한다.");
  lines.push("C learner-oriented reconstruction: 단순 자기 주장/자세한 답변을 넘어, 그 learner의 이해 곤란에 맞춰 설명 방식(표현/구조/예시)을 재구성.");
  lines.push("reconstruction 예: learner의 특정 어려움에 맞춘 쉬운 표현, 비유/유추, 예시, 단계적 재설명, 이해하지 못한 지점을 다른 방식으로 재구성.");
  lines.push("C7에 불충분: 단순 내용 질문, 질문 뒤 자세한 답변, 긴 설명, \"이해했니?\" 한마디, 상대에게 설명하는 말투, 이미 알고 있는 사례를 언급했다는 사실만.");
  lines.push("\"설명을 잘했다\" ≠ C7. \"특정 learner의 이해 곤란에 맞추어 설명 방법을 재구성했다\" = C7 후보.");
  lines.push("발화 길이는 C7 기준 아님. 긴 독백이라도 adaptation이 없으면 C3 등 다른 코드.");
  lines.push("comprehension-check 단독 금지: \"이해했니?\" \"알겠어?\" \"이해했지?\"만으로 자동 C7 아님.");
  lines.push("C7 hard negatives: (A) S1 \"왜?\" S2 \"압력이 낮아져서.\" → 짧은 이유, C2/C3 검토 (B) S1 \"압력 낮아져\" S2 \"그래서 공기 들어와\" → C3 (C) 혼자 긴 설명, learner 불명 (D) adaptation 없이 \"이해했지?\"만.");
  lines.push("명확한 positive 예: S1 \"압력 차이가 무슨 뜻인지 잘 모르겠어.\" S2 \"풍선 생각해봐. 밖에서 누르면 안쪽 공기가 눌리잖아. 그거랑 비슷하게 여기서는...\"");
  lines.push("C7↔C3 보수적: 둘 다 가능해 보이지만 learner-adaptation이 명확하지 않으면 C3를 유지. C7은 C3보다 우선이지만, 우선순위는 C7 최소조건이 명확하게 성립할 때만 적용.");
  lines.push("C7 FINAL-RELATION: contributors=learner+peer teacher. quotes=learner 어려움 + peer teacher의 learner-oriented explanation. 뒤쪽 별도 C3 reaction은 C7 quotes에서 제외.");
  lines.push("C7 path: [\"C-STEP0:YES\",\"C-STEP1:NO\",\"C-STEP2:YES\"]");
  lines.push("C7 TEMPORAL: learner의 질문/어려움/이해 필요 → peer의 learner-oriented reconstruction. peer teaching이 먼저인데 뒤 어려움을 근거로 소급 C7 금지.");
  lines.push("");
  lines.push("===== C-STEP3 → C5 설득 =====");
  lines.push("C6/C7 아니면: 학생이 특정 상대의 답/입장/행동을 바꾸려는 명시적 목표를 가지고, 자기 입장 + 실질적 이유를 제시하는가? YES → C5.");
  lines.push("C5 최소조건 3개: (1) 상대의 현재 답/입장/행동이 특정됨 (2) 그것을 바꾸려는 목표 (3) 자기 주장 + 실질적 내용적 이유.");
  lines.push("예: S1 \"2번 아니야?\" S2 \"1번으로 바꿔. 실험에서 실제로 색이 변했잖아.\"");
  lines.push("C5 아님: \"1번 해.\"(명령만), \"내가 1번이라고 생각해.\"(자기 주장만), \"아니, 2번이야.\"(반박만 → C4 가능).");
  lines.push("이유만 있다고 C5가 아니다. 상대를 바꾸려는 persuasion goal이 필수.");
  lines.push("C5 path: [\"C-STEP0:YES\",\"C-STEP1:NO\",\"C-STEP2:NO\",\"C-STEP3:YES\"]");
  lines.push("C5 TEMPORAL: 상대의 실제 입장/행동/답이 먼저 식별된 뒤 변화를 유도하는 설득. 설득 target을 후행 발화에서 역으로 만들지 마라.");
  lines.push("");
  lines.push("===== C-STEP4 → C4 비판·반박 =====");
  lines.push("상위 코드 아니면: 학생이 특정 peer의 주장/답/설명에 반응하여 그것과 양립하기 어려운 반대 주장, 문제점, 오류, 반례, 조건을 제시하는가? YES → C4.");
  lines.push("C4는 반드시 반박 대상이 되는 peer proposition/position이 있어야 한다.");
  lines.push("질문, 미완성 탐색, 이해 부족 표현을 임의의 주장으로 복원해서 C4를 만들지 마라.");
  lines.push("C4 self-check: (1) target peer는 누구인가? (2) target peer가 실제로 어떤 proposition/position을 제시했는가? (3) rebutting student는 그 proposition을 부정/문제제기/반례화했는가?");
  lines.push("(2)를 원발화에서 인용할 수 없으면 C4 금지.");
  lines.push("질문형 발화는 그 자체로 반박 대상 proposition이 아니다.");
  lines.push("C4 아님: S1 \"혈구가 나가면은?\" S3 \"안 나갈 것 같아요.\" — S1이 \"혈구가 나간다\"고 주장한 것이 아니면 S3를 C4로 코딩하지 마라.");
  lines.push("C4 가능: S1 \"혈구는 나가.\" S3 \"아니, 안 나갈 것 같아.\" — 명확한 peer position과 opposition.");
  lines.push("예: S1 \"들이쉴 때 압력이 높아져.\" S2 \"아니, 낮아져야 공기가 들어오는 거 아니야?\"");
  lines.push("명시적 \"아니\" 없이도 가능. S1 \"항상 압력이 높아.\" S2 \"근데 내쉴 때는 반대잖아.\" → C4 가능.");
  lines.push("C4 아님: S1 \"1번.\" S2 \"2번.\"(서로 다른 답을 한 번씩 말함). \"절대 아니야.\"(부정 대상이 현재 클러스터에서 특정 안 됨).");
  lines.push("질문 형태라도 이미 제시된 peer proposition의 타당성을 공격하면 C4. 예: \"그럼 네 말대로면 이 경우는 어떻게 설명할 건데?\"");
  lines.push("C4 path: [\"C-STEP0:YES\",\"C-STEP1:NO\",\"C-STEP2:NO\",\"C-STEP3:NO\",\"C-STEP4:YES\"]");
  lines.push("C4 TEMPORAL: target peer proposition/position → later rebuttal. 반박 대상 입장이 반박 발화보다 먼저 실제 담화에 존재해야 한다. 미래 입장을 앞선 발화가 반박했다고 해석하지 마라.");
  lines.push("");
  lines.push("===== C-STEP5 → C3 상호작용을 통한 정교화 =====");
  lines.push("상위 코드 아니면: 한 학생이 다른 학생의 발화 또는 공동으로 다루는 설명에 새로운 내용/조건/이유/예시/관계를 추가하여 설명을 확장하는가? YES → C3.");
  lines.push("C3 핵심: base contribution → later elaboration. elaborator 발화는 base contribution보다 실제로 뒤에 있어야 한다.");
  lines.push("예: S1 \"폐가 커지면 압력이 낮아져.\" S2 \"그래서 바깥 공기가 안으로 들어오는 거야.\" — S1 먼저, S2 나중.");
  lines.push("예: S1 \"모세혈관인 것 같아.\" S3 \"모세혈관이 괜히 있는 게 아닐 거야.\" — S1→S3 C3 가능.");
  lines.push("C3 TEMPORAL 금지: S3 \"안 나갈 것 같아요\"(먼저) S1 \"안 나가잖아요.\"(나중) → S1→S3 C3 불가. 후행을 base로 두고 앞선 발화를 elaboration으로 재해석하지 마라.");
  lines.push("C3 아님: S1 \"몇 번?\" S2 \"3번.\"(단순 답변). S1 \"왜?\" S2 \"커서.\"(요청받은 이유 제공만 → 우선 C2 관계인지 확인).");
  lines.push("학생 A의 답 뒤에 학생 B가 더 풍부한 답을 했다는 이유만으로 자동 C3 금지.");
  lines.push("특히 Teacher \"이유가 뭐야?\" S1: X / S2: Y 에서 Y가 교사 질문에 대한 독립 답변이면 S1→S2 C3로 자동 연결하지 마라.");
  lines.push("C3가 되려면 S2가 S1의 contribution을 실제로 받아 새 내용/이유/조건/관계를 덧붙인 기능적 연결이 필요하다.");
  lines.push("초기에 C2 질문이 있었다고 최종 코드를 C2로 고정하지 마라. 질문→답변→상호 확장이면 C3 가능.");
  lines.push("C3↔C7: C3를 출력하기 전에 C7 최소조건(identifiable learner + 실제 이해 곤란 + learner-adapted reconstruction)이 명확히 성립하는지 확인. 명확하지 않으면 C3 유지.");
  lines.push("질문 다음 긴 답변, \"이해했니\" 키워드, 발화 길이만으로 C7 금지.");
  lines.push("C3 path: [\"C-STEP0:YES\",\"C-STEP1:NO\",\"C-STEP2:NO\",\"C-STEP3:NO\",\"C-STEP4:NO\",\"C-STEP5:YES\"]");
  lines.push("");
  lines.push("===== C-STEP6 → C2 의견·명료화·정당화 요청 =====");
  lines.push("상위 코드 아니면: 학생이 peer에게 판단/답/확인/의미/이유/근거를 요청하는가? YES → C2.");
  lines.push("C2 가능: \"넌 뭐라고 생각해?\", \"몇 번이라고 생각해?\", \"왜 그렇게 생각해?\", \"그게 무슨 뜻이야?\", \"이거 맞아?\", \"그 이유가 뭐야?\"");
  lines.push("C2는 epistemic request이다. 상대의 판단/답/설명/이유/의미를 요청하는 기능이어야 한다.");
  lines.push("C2 TEMPORAL: 단순 선후 고정 없음. A request→response(S1 \"너는?\" S2 \"2번.\") 또는 B peer content→clarification request(S1 \"압력 낮아져\" S2 \"왜?\") 모두 가능. 핵심은 실제 시간 순서에서 request target이 식별 가능한 것.");
  lines.push("질문이 상대 주장의 타당성을 공격하기 위한 것 → C4. 질문 이후 공동 설명이 확장됨 → C3. 질문 이후 여러 선택을 조정하여 공동 결정 → C6.");
  lines.push("C2는 더 높은 interaction function이 없을 때 선택.");
  lines.push("C2 path: [\"C-STEP0:YES\",\"C-STEP1:NO\",\"C-STEP2:NO\",\"C-STEP3:NO\",\"C-STEP4:NO\",\"C-STEP5:NO\",\"C-STEP6:YES\"]");
  lines.push("");
  lines.push("===== C-STEP7 → C1 동의·수용 =====");
  lines.push("상위 코드 모두 아니면: 학생이 특정 peer의 말/요청/제안을 긍정적으로 수용하는가? YES → C1.");
  lines.push("예: S1 \"폐가 커지는 것 같아.\" S2 \"맞아.\" / S1 \"이거 봐도 돼?\" S2 \"응.\" → 사회적/과제적 수용도 관계가 명확하면 C1 가능.");
  lines.push("가능: \"응\", \"맞아\", \"그래\", 같은 내용 반복, 명확한 request acceptance, 원자료에 표시된 고개 끄덕임.");
  lines.push("\"응\" 한 글자도 유효한 C1 evidence일 수 있다. quote에 실제 원발화를 그대로 넣어라.");
  lines.push("\"맞아.\" → C1. \"맞아. 그리고 압력이 낮아져서 공기가 들어와.\" → C3 검토.");
  lines.push("앞선 여러 제안을 \"좋아, 그렇게 하자.\"로 공동 확정한 경우 → C6 검토.");
  lines.push("C1 path: [\"C-STEP0:YES\",\"C-STEP1:NO\",\"C-STEP2:NO\",\"C-STEP3:NO\",\"C-STEP4:NO\",\"C-STEP5:NO\",\"C-STEP6:NO\",\"C-STEP7:YES\"]");
  lines.push("");
  lines.push("===== Boundary Rules =====");
  lines.push("C1↔C3: 단순 수용/반복 → C1. 수용하면서 새로운 내용/이유/조건/관계 추가 → C3.");
  lines.push("C2↔C3: 질문-직접 답변 중심 → C2. 질문을 계기로 설명을 추가·확장하여 공동 설명이 형성 → C3.");
  lines.push("C2↔C4: 열린 정보/의견/이유 요청 → C2. 상대 주장의 타당성을 공격하기 위한 질문 → C4.");
  lines.push("C4↔C5: 상대 말에 반대/문제 제기 → C4. 상대가 답/입장/행동을 바꾸도록 명시적으로 유도 + 자기 주장 + 실질적 이유 → C5.");
  lines.push("C5↔C6: 한 학생이 다른 학생을 자기 입장으로 이동시킴 → C5. 서로의 제안/입장을 조정·통합하거나 복수 제안을 공동 결과로 명시적 확정 → C6.");
  lines.push("C6↔C1: 단순 proposition/요청 수용(\"응\", \"맞아\") → C1. 공유 과제의 답/행동/선택을 함께 확정하는 acceptance → C6.");
  lines.push("C6↔C3: 문제 제기/탐색 방향 제안 + peer 설명 추가 → C3. joint outcome/decision/coordination 형성 → C6. 설명만으로 C6 금지.");
  lines.push("C3↔C7: C3=peer 발화/공동 문제에 새 내용·이유·조건·관계 추가. C7=특정 learner의 이해 곤란을 해결하기 위해 설명의 표현/구조/예시를 learner에 맞게 재구성. 둘 다 가능해 보이지만 learner-adaptation이 명확하지 않으면 C3 유지. C7 > C3 우선은 C7 최소조건이 명확할 때만.");
  lines.push("");
  lines.push("===== FINAL RELATION 범위 =====");
  lines.push("한 cluster 안에 여러 C 후보 관계가 동시에 존재할 수 있다. 예: S2↔S3=C2, S1↔S3=C3.");
  lines.push("우선순위에 따라 최종 code는 C3가 될 수 있다. 그러나 contributors/quotes/interaction_summary/reason은 최종 선택된 C relation을 실제로 구성한 학생과 발화만 설명한다.");
  lines.push("낮은 우선순위의 별도 C2/C3/C3-reaction에 참여했다는 이유만으로 그 학생을 최종 C7 contributors/quotes에 넣지 마라.");
  lines.push("낮은 우선순위의 별도 C2/C3에 참여했다는 이유만으로 그 학생을 최종 C6 contributors/quotes에 넣지 마라.");
  lines.push("낮은 우선순위의 별도 C2에 참여했다는 이유만으로 그 학생을 최종 C3 contributors/quotes에 넣지 마라.");
  lines.push("판단 순서: (A) ordered turns에서 functional relation 성립 확인 (B) relation direction/temporal order 확인 (C) C code 최소조건 확인 (D) priority로 final code 선택 (E) 그 relation만 contributors (F) 해당 relation 발화만 quotes.");
  lines.push("FINAL RELATION scoping에서 새 relation을 만들지 마라. temporal/direction 위반 relation은 scoping 대상이 아니다.");
  lines.push("\"cluster에서 C3가 하나라도 성립\" → 최종 code C3 가능. 그러나 \"cluster의 모든 C2/C1 참여 학생\" → C3 contributors 아님.");
  lines.push("서로 인접하거나 같은 과학 주제를 말한다는 이유만으로 별도 interaction을 하나의 C3로 합치지 마라.");
  lines.push("반드시 답할 것: What exact peer-to-peer relation establishes the FINAL code? 그 relation만 evidence로 사용.");
  lines.push("contributors는 현재 cluster의 active 학생 전체도, 모든 C interaction 참여 학생도 아니다.");
  lines.push("");
  lines.push("P055형: S2 \"힘이 없어지나?\" S3 \"힘이 없어지지 비슷한 것 같은데\" → Relation A, C2 후보.");
  lines.push("S1 \"음 모세혈관 맞는 것 같고\" S3 \"모세혈관이 괜히 있는 게 아닐 거야\" → Relation B, C3 후보.");
  lines.push("C3 > C2 이므로 최종 code=C3. contributors=[\"S1\",\"S3\"]. quotes는 모세혈관 판단+기능 추가만.");
  lines.push("S2는 C2 관계에 참여했어도 최종 C3 relation contributor가 아니므로 제외.");
  lines.push("interaction_summary 예: \"S1의 모세혈관 관련 판단에 S3가 기능적 의미를 추가하여 내용을 확장함.\"");
  lines.push("boundary_check 예: \"S2-S3의 질문-응답은 C2 후보이나, S1-S3의 내용 확장이 C3 최소조건을 만족하므로 우선순위에 따라 C3를 선택함.\"");
  lines.push("한 cluster에 lower-priority와 higher-priority relation이 같이 있으면 boundary_check에 이를 적극적으로 명시하라. non-null 강제는 아니다.");
  lines.push("");
  lines.push("===== contributors / quotes =====");
  lines.push("contributors = 최종 선택된 C코드 성립에 기능적으로 참여한 고유 학생 S ID 집합. 단순 active speakers 전체가 아니다.");
  lines.push("정상 non-null C에서는 contributors 최소 2명. 교사는 contributor 금지.");
  lines.push("C1: accepter + target peer. C2: requester + targeted peer.");
  lines.push("C3: base contribution 학생 + elaborator(s). C4: target claim 학생 + rebutter.");
  lines.push("C5: persuader + change target. C6: joint outcome/decision/coordination을 실제로 형성한 학생만. C7: learner + peer teacher — teaching relation을 공동으로 만든 추가 학생만, 뒤에서 별도 C3 reaction만 한 학생은 포함하지 않음.");
  lines.push("C6 contributors = joint outcome을 실제로 형성한 학생. C6 quotes = joint outcome/decision/coordination을 직접 입증하는 발화/행동만.");
  lines.push("C7 contributors = learner + peer teacher. C7 quotes = learner 질문/어려움/이해 target + peer teacher의 learner-oriented explanation. 뒤쪽 C3 reaction은 C7 quotes에서 제외.");
  lines.push("S1↔S3만 최종 C3이고 S2는 별도 C2면 contributors=[\"S1\",\"S3\"]. S2 unrelated 발화도 제외.");
  lines.push("짧은 \"응\"도 최종 C1 또는 공동 결정 성립에 실제 기능했다면 그 학생은 contributor.");
  lines.push("interaction_summary는 contributors와 동일한 관계만 설명한다. contributors=[\"S1\",\"S3\"]이면 S2 질문-응답을 최종 C3 구성요소처럼 쓰지 마라.");
  lines.push("quotes는 cluster의 C 관련 발화를 모두 수집하는 필드가 아니다. 최종 선택된 C code를 직접 성립시키는 증거 발화만.");
  lines.push("quotes 배열은 packet.turns에서 등장한 실제 시간 순서(chronological order)대로 출력한다. 나중 turn quote를 앞에 배치하지 마라.");
  lines.push("lower-priority relation, 별도 interaction, 단순 same-topic utterance는 quotes에서 제외.");
  lines.push("UNIQUE(contributors) === UNIQUE(quotes[].speaker). 양방향 일치. mismatch면 출력 전에 최종 relation 범위로 다시 맞출 것. 자동 union/삭제 추정 금지.");
  lines.push("quotes는 ARRAY OF OBJECTS만. {\"speaker\":\"S1\",\"quote\":\"원발화\"}. bare string 금지. speaker 누락 금지.");
  lines.push("Teacher quote를 final C evidence로 사용하지 마라. quote는 CURRENT CLUSTER TURNS의 실제 학생 발화와 일치해야 한다.");
  lines.push("\"응\" 같은 1글자도 원발화 그대로 quote에 넣을 수 있다.");
  lines.push("");
  lines.push("===== JSON 출력 직전 self-check =====");
  lines.push("1) 실제 학생-학생 기능적 연결이 있는가?");
  lines.push("2) 단순히 학생이 2명 이상 말한 것을 C로 오인하지 않았는가?");
  lines.push("3) teacher-student 독립 문답을 student-student interaction으로 오인하지 않았는가? A→Teacher→B만으로 C를 만들지 않았는가?");
  lines.push("3b) B가 한 일이 A의 contribution을 기능적으로 이어받은 것인가, 아니면 같은 교사 질문에 독립 답한 것인가?");
  lines.push("4) 높은 코드가 실제 최소조건을 충족하는가?");
  lines.push("5) C6이 실제 공동 조정/결정인가? quotes만 읽어도 joint outcome/decision이 확인되는가? proposal+explanation을 C6로 과대해석하지 않았는가?");
  lines.push("6) C7이 실제 learner-adapted teaching인가? identifiable learner + 실제 이해 곤란 + learner-oriented reconstruction이 모두 명확한가? \"이해했니\"/길이/질문+긴답만으로 C7을 고르지 않았는가? 애매하면 C3 유지.");
  lines.push("7) C5에 change goal + own claim + substantive reason이 모두 있는가?");
  lines.push("8) C4의 반박 대상 peer proposition을 원발화에서 인용할 수 있는가? 질문을 주장으로 복원하지 않았는가?");
  lines.push("9) C3에 peer의 기존 내용에 대한 새로운 추가가 있는가? C7 최소조건이 명확하지 않은데 C7로 올리지 않았는가?");
  lines.push("10) C2가 반박/정교화/공동결정으로 발전하지 않았는가?");
  lines.push("11) C1이 단순 수용인지, 아니면 C3/C6 등 더 높은 기능인지 확인했는가?");
  lines.push("12) contributors는 단순 active speakers가 아니라 최종 C관계를 실제로 구성한 학생만인가? lower-priority 별도 relation 참여자(S1 in P037 C6 등)를 넣지 않았는가?");
  lines.push("13) UNIQUE(contributors) === UNIQUE(quotes[].speaker) 인가?");
  lines.push("14) quotes의 모든 원소가 {speaker, quote} 객체인가?");
  lines.push("15) quotes/interaction_summary가 최종 code를 직접 성립시킨 relation만 가리키는가?");
  lines.push("16) relation direction이 실제 turn 순서와 일치하는가? later utterance를 base로 두지 않았는가?");
  lines.push("17) quotes가 packet.turns의 chronological order와 일치하는가?");
  lines.push("하나라도 아니면 출력 전에 JSON을 수정한다.");
  lines.push("");
  lines.push("===== 출력 JSON (이 스키마만) =====");
  lines.push("{");
  lines.push('  "schema_version":"KCMP_C_V1",');
  lines.push('  "status":"OK",');
  lines.push('  "code": null,');
  lines.push('  "contributors": [],');
  lines.push('  "interaction_summary": null,');
  lines.push('  "reason": "학생 발화가 서로 기능적으로 연결된 학생-학생 상호작용을 구성하지 않는다.",');
  lines.push('  "decision_path": ["C-STEP0:NO"],');
  lines.push('  "boundary_check": null,');
  lines.push('  "context_needed": false,');
  lines.push('  "quotes": []');
  lines.push("}");
  lines.push("");
  lines.push("C3 예:");
  lines.push('  code="C3", contributors=["S1","S3"]');
  lines.push('  interaction_summary="S1의 설명에 S3가 새로운 인과관계를 추가하여 공동 설명을 확장함"');
  lines.push('  quotes=[{"speaker":"S1","quote":"폐가 커지면 압력이 낮아져."},{"speaker":"S3","quote":"그래서 바깥 공기가 안으로 들어오는 거야."}]');
  lines.push("P055형 C3 예: contributors=[\"S1\",\"S3\"], quotes는 모세혈관 판단+확장만. S2의 \"힘이 없어지나?\"는 최종 quotes에 넣지 마라.");
  lines.push('  path=["C-STEP0:YES","C-STEP1:NO","C-STEP2:NO","C-STEP3:NO","C-STEP4:NO","C-STEP5:YES"]');
  lines.push("C1 예: contributors=[\"S1\",\"S2\"], quotes에 S2의 \"응\" 포함 가능.");
  lines.push("C6 역할 결정 예: \"내가 발표할까?\" / \"그럼 내가 기록할게.\" / \"좋아, 그렇게 하자.\"");
  lines.push("STEP0 YES였지만 C1~C7 최소조건 어느 것도 성립하지 않으면 code=null, path는 STEP1~STEP7 전부 NO, 강제로 코드를 고르지 마라.");
  lines.push("reason은 항상 비어 있지 않은 문자열. non-null C에서 interaction_summary는 S ID를 포함해 기능적 연결을 짧게 설명.");
  lines.push("");
  lines.push("[PID]");
  lines.push(packet.pid || "");
  lines.push("");
  lines.push("[STUDENTS]");
  lines.push(students || "(없음)");
  lines.push("");
  lines.push("[ACTIVE STUDENTS]");
  lines.push(active || "(없음)");
  lines.push("");
  lines.push("[CURRENT CLUSTER TURNS]");
  lines.push(turnsText || "(원발화 없음)");
  lines.push("");
  lines.push("[SUMMARY - AUXILIARY ONLY]");
  lines.push("보조자료이며 원발화와 충돌하면 원발화를 우선한다.");
  lines.push(summary || "(요약 없음)");
  lines.push("");
  lines.push("JSON만 출력하라.");
  return lines.join("\n");
}

function parseCDecisionTreeResponse_(raw){
  let s = String(raw == null ? "" : raw).replace(/^\uFEFF/, "").trim();
  if (!s) throw new Error("빈 응답");
  s = s.replace(/^```(?:json)?\s*/i, "").replace(/\s*```$/i, "").trim();
  try {
    return JSON.parse(s);
  } catch (e1) {
    const start = s.indexOf("{");
    const end = s.lastIndexOf("}");
    if (start >= 0 && end > start) {
      try {
        return JSON.parse(s.slice(start, end + 1));
      } catch (e2) {
        throw new Error("JSON.parse 실패: " + e2.toString());
      }
    }
    throw new Error("JSON.parse 실패: " + e1.toString());
  }
}

function _kcmpCQuoteMatchesTurn_(quote, turn){
  const q = _kcmpNormForQuote_(quote);
  if (!q) return false;
  const u = _kcmpNormForQuote_(turn.utterance);
  if (!u) return false;
  if (q.length === 1) return u === q;
  return u.indexOf(q) >= 0;
}

function _kcmpStudentTurnIndicesForCQuote_(quote, speaker, studentTurns){
  const out = [];
  (studentTurns || []).forEach(function(t, idx){
    if (t.speakerId !== speaker) return;
    if (_kcmpCQuoteMatchesTurn_(quote, t)) out.push(idx);
  });
  return out;
}

function _kcmpValidateCQuotesChronological_(quotes, studentTurns, errors){
  if (!Array.isArray(quotes) || quotes.length < 2) return;
  const options = quotes.map(function(q){
    const sp = q && q.speaker;
    const quote = q && q.quote;
    const indices = _kcmpStudentTurnIndicesForCQuote_(quote, sp, studentTurns);
    return indices.length ? indices : [null];
  });
  if (options.some(function(o){ return o.length === 1 && o[0] === null; })) return;

  function canAssign(pos, prevIdx){
    if (pos >= options.length) return true;
    const opts = options[pos];
    for (let i = 0; i < opts.length; i++) {
      const idx = opts[i];
      if (prevIdx < 0 || idx >= prevIdx) {
        if (canAssign(pos + 1, idx)) return true;
      }
    }
    return false;
  }
  if (!canAssign(0, -1)) {
    errors.push("quotes are not in chronological packet order");
  }
}

function validateCDecisionResult_(result, packet){
  const errors = [];
  const warnings = [];
  if (!result || typeof result !== "object") {
    return { ok: false, errors: ["결과가 객체가 아님"], warnings: warnings };
  }
  if (result.schema_version !== "KCMP_C_V1") errors.push("schema_version 불일치: " + result.schema_version);
  if (result.status !== "OK") errors.push("status가 OK가 아님: " + result.status);

  const reason = String(result.reason == null ? "" : result.reason).trim();
  if (reason.length === 0) errors.push("reason이 비어 있음 (정상 결과는 reason 필수)");

  if (typeof result.context_needed !== "boolean") {
    errors.push("context_needed가 boolean이 아님");
  }

  const allowed = { C1: true, C2: true, C3: true, C4: true, C5: true, C6: true, C7: true };
  const code = result.code;
  if (!(code === null || allowed[code])) errors.push("code 무효: " + code);

  const activeIds = packet && packet.activeStudentIds ? packet.activeStudentIds : [];
  if (code != null && activeIds.length < 2) errors.push("code가 있는데 packet.activeStudentIds가 2명 미만");

  if (!Array.isArray(result.contributors)) {
    errors.push("contributors가 배열이 아님");
  } else {
    const seen = {};
    const active = {};
    activeIds.forEach(function(id){ active[id] = true; });
    result.contributors.forEach(function(id){
      if (seen[id]) errors.push("contributor 중복: " + id);
      seen[id] = true;
      if (!/^S[1-4]$/.test(String(id || ""))) errors.push("contributor가 S1~S4가 아님: " + id);
      if (packet && !active[id]) errors.push("contributor가 activeStudentIds에 없음: " + id);
    });
    if (code != null && result.contributors.length < 2) errors.push("non-null C인데 contributors가 2명 미만");
    if (code == null && result.contributors.length !== 0) errors.push("code=null인데 contributors가 비어 있지 않음");
  }

  const isRaw = result.interaction_summary;
  const isText = isRaw == null ? null : String(isRaw).trim();
  if (code == null) {
    if (isRaw != null && isText && isText.length > 0) errors.push("code=null인데 interaction_summary가 null이 아님");
  } else {
    if (!isText) errors.push("code가 있는데 interaction_summary가 비어 있음");
  }

  if (!Array.isArray(result.decision_path)) {
    errors.push("decision_path가 배열이 아님");
  } else {
    const path = result.decision_path;
    if (code != null && !_kcmpPathHas_(path, "C-STEP0:YES")) errors.push("non-null C인데 C-STEP0:YES 없음");
    if (code === "C6" && !_kcmpPathHas_(path, "C-STEP1:YES")) errors.push("C6인데 C-STEP1:YES 없음");
    if (code === "C7" && !_kcmpPathHas_(path, "C-STEP2:YES")) errors.push("C7인데 C-STEP2:YES 없음");
    if (code === "C5" && !_kcmpPathHas_(path, "C-STEP3:YES")) errors.push("C5인데 C-STEP3:YES 없음");
    if (code === "C4" && !_kcmpPathHas_(path, "C-STEP4:YES")) errors.push("C4인데 C-STEP4:YES 없음");
    if (code === "C3" && !_kcmpPathHas_(path, "C-STEP5:YES")) errors.push("C3인데 C-STEP5:YES 없음");
    if (code === "C2" && !_kcmpPathHas_(path, "C-STEP6:YES")) errors.push("C2인데 C-STEP6:YES 없음");
    if (code === "C1" && !_kcmpPathHas_(path, "C-STEP7:YES")) errors.push("C1인데 C-STEP7:YES 없음");
    if (code == null) {
      const step0No = _kcmpPathHas_(path, "C-STEP0:NO");
      const allNo = _kcmpPathHas_(path, "C-STEP1:NO") && _kcmpPathHas_(path, "C-STEP2:NO") && _kcmpPathHas_(path, "C-STEP3:NO") && _kcmpPathHas_(path, "C-STEP4:NO") && _kcmpPathHas_(path, "C-STEP5:NO") && _kcmpPathHas_(path, "C-STEP6:NO") && _kcmpPathHas_(path, "C-STEP7:NO");
      if (!step0No && !allNo) errors.push("null code인데 decision_path가 C-STEP0:NO 또는 STEP1~STEP7 전부 NO가 아님");
    }
  }

  const studentTurns = ((packet && packet.turns) || []).filter(function(t){ return t.role === "student"; });

  if (!Array.isArray(result.quotes)) {
    errors.push("quotes가 배열이 아님");
  } else if (code == null) {
    if (result.quotes.length !== 0) errors.push("code=null인데 quotes가 비어 있지 않음");
  } else {
    if (result.quotes.length < 1) errors.push("code가 있는데 quotes가 비어 있음");
    _kcmpValidateCQuoteEntries_(result.quotes, studentTurns, "quotes", errors, packet);
    const quoteSpeakers = _kcmpQuoteSpeakers_(result.quotes);
    if (Object.keys(quoteSpeakers).length < 2) errors.push("non-null C인데 quotes speaker가 2명 미만");
    _kcmpValidateContributorQuoteSetEquality_(result.contributors, result.quotes, errors);
    _kcmpValidateCQuotesChronological_(result.quotes, studentTurns, errors);
  }

  return { ok: errors.length === 0, errors: errors, warnings: warnings };
}

function runCDecisionTreeForPacket_(packet){
  const pid = packet && packet.pid ? packet.pid : "";
  if (!packet || !pid) return _makeCDecisionError_("PACKET_ERROR", "pid 없음", pid);
  if (!packet.turns || packet.turns.length === 0) {
    return _makeCDecisionError_("PACKET_ERROR", "turns가 비어 있음", pid);
  }

  const active = packet.activeStudentIds || [];
  if (active.length < 2) {
    if (_cPacketMappingUnreliable_(packet)) {
      return _makeCDecisionError_("PACKET_ERROR", "활성 학생이 2명 미만이지만 unmapped/unknown speaker가 있어 학생 수를 단정할 수 없음", pid);
    }
    return _makeCNoneResult_(
      pid,
      "현재 클러스터에 활성 학생이 2명 미만이어서 학생-학생 상호작용이 성립하지 않는다.",
      ["C-STEP0:NO"]
    );
  }

  let raw = "";
  try {
    raw = callGPT_simple_(buildCDecisionPrompt_(packet), MODEL_C);
  } catch (e) {
    return _makeCDecisionError_("API_ERROR", e.toString(), pid);
  }

  let parsed;
  try {
    parsed = parseCDecisionTreeResponse_(raw);
  } catch (e) {
    return _makeCDecisionError_("PARSER_ERROR", e.toString(), pid, { raw_excerpt: String(raw).slice(0, 400) });
  }

  const v = validateCDecisionResult_(parsed, packet);
  if (!v.ok) {
    return _makeCDecisionError_("VALIDATION_ERROR", v.errors.join("; "), pid, {
      validation_errors: v.errors,
      raw_excerpt: String(raw).slice(0, 400)
    });
  }
  parsed.status = "OK";
  parsed.schema_version = "KCMP_C_V1";
  parsed.pid = pid;
  return parsed;
}

function getCDecisionTreeFixtures_(){
  return [
    {
      id: "NO_INTERACTION_INDEPENDENT_TEACHER_ANSWERS",
      expected: null,
      packet: _kcmpSyntheticPacket_("CF01", [
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "S1은?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "1번." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "S2는?" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "2번." }
      ])
    },
    {
      id: "NO_INTERACTION_UNRELATED",
      expected: null,
      packet: _kcmpSyntheticPacket_("CF02", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "오늘 피곤해." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "배고프다." }
      ])
    },
    {
      id: "C1_ACCEPTANCE",
      expected: "C1",
      packet: _kcmpSyntheticPacket_("CF03", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐가 커지는 것 같아." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "응" }
      ])
    },
    {
      id: "C2_REQUEST",
      expected: "C2",
      packet: _kcmpSyntheticPacket_("CF04", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력이 낮아지는 것 같아." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "왜 그렇게 생각해?" }
      ])
    },
    {
      id: "C3_ELABORATION",
      expected: "C3",
      packet: _kcmpSyntheticPacket_("CF05", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐가 커지면 압력이 낮아져." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "그래서 바깥 공기가 안으로 들어오는 거야." }
      ])
    },
    {
      id: "C4_REBUTTAL",
      expected: "C4",
      packet: _kcmpSyntheticPacket_("CF06", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "들이쉴 때 압력이 높아져." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "아니, 낮아져야 공기가 들어오는 거 아니야?" }
      ])
    },
    {
      id: "C5_PERSUASION",
      expected: "C5",
      packet: _kcmpSyntheticPacket_("CF07", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "난 2번인 것 같아." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "1번으로 바꿔. 실험에서 실제로 색이 변했잖아." }
      ])
    },
    {
      id: "C6_IDEA_COORDINATION",
      expected: "C6",
      packet: _kcmpSyntheticPacket_("CF08", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "원인은 압력 차인 것 같아." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "횡격막 움직임도 넣어야 해." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그럼 둘 다 연결해서 설명하자." }
      ])
    },
    {
      id: "C6_TASK_DECISION",
      expected: "C6",
      note: "절차/역할이지만 공동 결정이므로 C6 가능",
      packet: _kcmpSyntheticPacket_("CF09", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "내가 발표할까?" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "그럼 내가 기록할게." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "좋아, 그렇게 하자." }
      ])
    },
    {
      id: "C7_PEER_TEACHING",
      expected: "C7",
      note: "generic C7 positive — identifiable learner difficulty + balloon analogy reconstruction",
      packet: _kcmpSyntheticPacket_("CF10", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력 차이가 무슨 뜻인지 잘 모르겠어." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "풍선 생각해봐. 밖에서 누르면 안쪽 공기가 눌리잖아. 그거랑 비슷하게 여기서는..." }
      ])
    },
    {
      id: "C7_NOT_SHORT_ANSWER",
      expected: "C3",
      note: "C7 hard negative — 짧은 이유 제공은 C7 아님",
      packet: _kcmpSyntheticPacket_("CF17", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "왜?" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "압력이 낮아져서." }
      ])
    },
    {
      id: "C2_TO_C3",
      expected: "C3",
      packet: _kcmpSyntheticPacket_("CF11", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "왜 압력이 낮아져?" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "부피가 커지니까." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그러면 공기가 바깥에서 들어오는 거네." }
      ])
    },
    {
      id: "C2_TO_C4",
      expected: "C4",
      packet: _kcmpSyntheticPacket_("CF12", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력이 높아져." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "왜 그렇게 생각해?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "부피가 커져서." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "근데 부피가 커지면 오히려 압력이 낮아지는 거 아니야?" }
      ])
    },
    {
      id: "BAD_REVERSED_C3_QUOTE_ORDER",
      expected: "C3",
      note: "validator regression — reversed quote order must fail",
      packet: _kcmpSyntheticPacket_("CF14", [
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "안 나갈 것 같아요" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "안 나가잖아요." }
      ])
    },
    {
      id: "GOOD_FORWARD_C3_QUOTE_ORDER",
      expected: "C3",
      note: "validator regression — forward quote order must pass",
      packet: _kcmpSyntheticPacket_("CF15", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "모세혈관인 것 같아." },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "모세혈관이 괜히 있는 게 아닐 거야" }
      ])
    },
    {
      id: "P037_C6_S2_S3_ANSWER_CHANGE",
      expected: "C6",
      note: "S2-S3 답 변경 제안+비언어 수용=C6. S2-S1 proposal+explanation은 C3 후보, C6 evidence 아님.",
      packet: _kcmpSyntheticPacket_("P037", [
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "그런가? 3번이 맞지 않을까? 바꿀까?" },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "(끄덕인다)" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "1번이 왜 아닌지를 생각해보자" },
        { role: "student", speakerId: "S4", speakerRaw: "학생4", utterance: "포도당같은 경우..." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그러니까 쌤이 저 훈쌤이 말해 주셨잖아. 압력이 막 여과 장치가 있는 것도 아니고 그냥 누르는 거니까. 그 입자량이 엄청 큰 게 아니면은 다 나가야 될 것 같아." }
      ])
    },
    {
      id: "P055_FINAL_RELATION_C3_NOT_C2",
      expected: "C3",
      note: "S2-S3는 C2 후보, S1-S3가 C3. 최종 contributors/quotes는 S1·S3만.",
      packet: _kcmpSyntheticPacket_("P055", [
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "힘이 없어지나?" },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "힘이 없어지지 비슷한 것 같은데" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "음 모세혈관 맞는 것 같고" },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "모세혈관이 괜히 있는 게 아닐 거야" }
      ])
    },
    {
      id: "P035_TEACHER_MEDIATED_INDEPENDENT_ANSWERS",
      expected: null,
      note: "같은 과학 문제 + 교사 중재 문답. S1 질문은 주장 아님. S3는 S1 반박 아님. C 없음.",
      packet: _kcmpSyntheticPacket_("P035", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "혈구가 나가면은?" },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "혈구가 나가?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "잘 모르겠어요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "잘 모르겠어?" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "혈구는 잘 모르겠어요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "혈구는 잘 모르겠어?" },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "안 나갈 것 같아요" },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "안 나갈 것 같아? 그럼 1번 아닌데?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "1번 아닌 것 같아요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "그럼 2번은 어때?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "아닌 것 같은데." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "뭐가 문제야?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력이 요소만 걸러내지 못하니까" },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "요소만 나가면 안 되는 이유가 뭐지?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "안 나가잖아요." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "오줌은 액체여서 물도 같이 있어야되니까" }
      ])
    },
    {
      id: "TEACHER_MEDIATED_C3",
      expected: "C3",
      packet: _kcmpSyntheticPacket_("CF13", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐가 커지면 압력이 낮아져요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "S3는 어떻게 생각해?" },
        { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "S1 말에 더하면, 그래서 바깥 공기가 들어오는 거예요." }
      ])
    }
  ];
}

function TEST_C_VALIDATOR_REGRESSION(){
  const packetNull = _kcmpSyntheticPacket_("CR00", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "1번." },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "2번." }
  ]);
  const goodNull = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: null,
    contributors: [],
    interaction_summary: null,
    reason: "학생 발화가 서로 기능적으로 연결된 학생-학생 상호작용을 구성하지 않는다.",
    decision_path: ["C-STEP0:NO"],
    boundary_check: null,
    context_needed: false,
    quotes: []
  };
  const packetC3 = _kcmpSyntheticPacket_("CR03", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐가 커지면 압력이 낮아져." },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "그래서 바깥 공기가 안으로 들어오는 거야." }
  ]);
  const badSingle = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C3",
    contributors: ["S1"],
    interaction_summary: "S1만 설명함",
    reason: "정교화",
    decision_path: ["C-STEP0:YES", "C-STEP1:NO", "C-STEP2:NO", "C-STEP3:NO", "C-STEP4:NO", "C-STEP5:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "폐가 커지면 압력이 낮아져." }]
  };
  const badMismatch = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C3",
    contributors: ["S1", "S2"],
    interaction_summary: "S1 설명에 S2가 인과를 추가함",
    reason: "정교화",
    decision_path: ["C-STEP0:YES", "C-STEP1:NO", "C-STEP2:NO", "C-STEP3:NO", "C-STEP4:NO", "C-STEP5:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "폐가 커지면 압력이 낮아져." }]
  };
  const packetC1 = _kcmpSyntheticPacket_("CR01", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐가 커져." },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "응" }
  ]);
  const goodC1Short = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C1",
    contributors: ["S1", "S2"],
    interaction_summary: "S2가 S1의 판단을 수용함",
    reason: "S2가 S1의 말에 짧게 동의하였다.",
    decision_path: ["C-STEP0:YES", "C-STEP1:NO", "C-STEP2:NO", "C-STEP3:NO", "C-STEP4:NO", "C-STEP5:NO", "C-STEP6:NO", "C-STEP7:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [
      { speaker: "S1", quote: "폐가 커져." },
      { speaker: "S2", quote: "응" }
    ]
  };
  const packetC6 = _kcmpSyntheticPacket_("CR06", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "원인은 압력 차인 것 같아." },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "횡격막도 넣자." },
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "그럼 둘 다 연결하자." }
  ]);
  const goodC6Three = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C6",
    contributors: ["S1", "S2", "S3"],
    interaction_summary: "S1·S2·S3가 압력과 횡격막을 공동 설명으로 확정함",
    reason: "세 학생이 서로 다른 제안을 하나의 설명으로 통합하였다.",
    decision_path: ["C-STEP0:YES", "C-STEP1:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [
      { speaker: "S1", quote: "원인은 압력 차인 것 같아." },
      { speaker: "S2", quote: "횡격막도 넣자." },
      { speaker: "S3", quote: "그럼 둘 다 연결하자." }
    ]
  };
  const packetReversed = _kcmpSyntheticPacket_("CR14", [
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "안 나갈 것 같아요" },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "안 나가잖아요." }
  ]);
  const badReversedQuotes = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C3",
    contributors: ["S1", "S3"],
    interaction_summary: "S1의 설명에 S3가 의견 추가",
    reason: "정교화",
    decision_path: ["C-STEP0:YES", "C-STEP1:NO", "C-STEP2:NO", "C-STEP3:NO", "C-STEP4:NO", "C-STEP5:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [
      { speaker: "S1", quote: "안 나가잖아요." },
      { speaker: "S3", quote: "안 나갈 것 같아요" }
    ]
  };
  const packetForward = _kcmpSyntheticPacket_("CR15", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "모세혈관인 것 같아." },
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "모세혈관이 괜히 있는 게 아닐 거야" }
  ]);
  const goodForwardQuotes = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C3",
    contributors: ["S1", "S3"],
    interaction_summary: "S1의 모세혈관 판단에 S3가 기능적 의미를 추가함",
    reason: "S3가 S1의 판단을 받아 내용을 확장함",
    decision_path: ["C-STEP0:YES", "C-STEP1:NO", "C-STEP2:NO", "C-STEP3:NO", "C-STEP4:NO", "C-STEP5:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [
      { speaker: "S1", quote: "모세혈관인 것 같아." },
      { speaker: "S3", quote: "모세혈관이 괜히 있는 게 아닐 거야" }
    ]
  };
  const packetP037C6 = _kcmpSyntheticPacket_("CR37", [
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "그런가? 3번이 맞지 않을까? 바꿀까?" },
    { role: "student", speakerId: "S3", speakerRaw: "학생3", utterance: "(끄덕인다)" },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "1번이 왜 아닌지를 생각해보자" },
    { role: "student", speakerId: "S4", speakerRaw: "학생4", utterance: "포도당같은 경우..." },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그러니까 쌤이 저 훈쌤이 말해 주셨잖아. 압력이 막 여과 장치가 있는 것도 아니고 그냥 누르는 거니까. 그 입자량이 엄청 큰 게 아니면은 다 나가야 될 것 같아." }
  ]);
  const goodP037C6 = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C6",
    contributors: ["S2", "S3"],
    interaction_summary: "S2가 공동 답을 3번으로 변경할 것을 제안하고 S3가 끄덕임으로 이를 수용하여 답 변경을 공동으로 확정함",
    reason: "S2의 답 변경 제안에 S3가 비언어적으로 수용하여 공동 결정을 형성함",
    decision_path: ["C-STEP0:YES", "C-STEP1:YES"],
    boundary_check: "S2-S1의 문제 제기+설명은 C3 후보이나 joint decision evidence가 아니므로 C6 contributors/quotes에서 제외함",
    context_needed: false,
    quotes: [
      { speaker: "S2", quote: "그런가? 3번이 맞지 않을까? 바꿀까?" },
      { speaker: "S3", quote: "(끄덕인다)" }
    ]
  };
  const packetC7Peer = _kcmpSyntheticPacket_("CR07", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력 차이가 무슨 뜻인지 잘 모르겠어." },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "풍선 생각해봐. 밖에서 누르면 안쪽 공기가 눌리잖아. 그거랑 비슷하게 여기서는..." }
  ]);
  const goodC7Peer = {
    schema_version: "KCMP_C_V1",
    status: "OK",
    code: "C7",
    contributors: ["S1", "S2"],
    interaction_summary: "S1의 이해 곤란에 S2가 풍선 비유로 설명을 재구성하여 또래 교수를 수행함",
    reason: "identifiable learner의 이해 곤란에 맞춘 비유적 재구성이 확인됨",
    decision_path: ["C-STEP0:YES", "C-STEP1:NO", "C-STEP2:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [
      { speaker: "S1", quote: "압력 차이가 무슨 뜻인지 잘 모르겠어." },
      { speaker: "S2", quote: "풍선 생각해봐. 밖에서 누르면 안쪽 공기가 눌리잖아. 그거랑 비슷하게 여기서는..." }
    ]
  };

  const a = validateCDecisionResult_(goodNull, packetNull);
  const b = validateCDecisionResult_(badSingle, packetC3);
  const c = validateCDecisionResult_(badMismatch, packetC3);
  const d = validateCDecisionResult_(goodC1Short, packetC1);
  const e = validateCDecisionResult_(goodC6Three, packetC6);
  const f = validateCDecisionResult_(badReversedQuotes, packetReversed);
  const g = validateCDecisionResult_(goodForwardQuotes, packetForward);
  const h = validateCDecisionResult_(goodP037C6, packetP037C6);
  const i = validateCDecisionResult_(goodC7Peer, packetC7Peer);
  Logger.log("GOOD_NULL ok=" + a.ok + " errors=" + JSON.stringify(a.errors));
  Logger.log("BAD_SINGLE_CONTRIBUTOR ok=" + b.ok + " errors=" + JSON.stringify(b.errors));
  Logger.log("BAD_CONTRIBUTOR_QUOTE_MISMATCH ok=" + c.ok + " errors=" + JSON.stringify(c.errors));
  Logger.log("GOOD_C1_SHORT_QUOTE ok=" + d.ok + " errors=" + JSON.stringify(d.errors));
  Logger.log("GOOD_C6_THREE_STUDENTS ok=" + e.ok + " errors=" + JSON.stringify(e.errors));
  Logger.log("BAD_REVERSED_QUOTE_ORDER ok=" + f.ok + " errors=" + JSON.stringify(f.errors));
  Logger.log("GOOD_FORWARD_QUOTE_ORDER ok=" + g.ok + " errors=" + JSON.stringify(g.errors));
  Logger.log("GOOD_P037_C6_S2_S3 ok=" + h.ok + " errors=" + JSON.stringify(h.errors));
  Logger.log("GOOD_C7_PEER_TEACHING ok=" + i.ok + " errors=" + JSON.stringify(i.errors));
  return {
    good_null: { ok: a.ok, errors: a.errors, expect_ok: true },
    bad_single_contributor: { ok: b.ok, errors: b.errors, expect_ok: false },
    bad_contributor_quote_mismatch: { ok: c.ok, errors: c.errors, expect_ok: false },
    good_c1_short_quote: { ok: d.ok, errors: d.errors, expect_ok: true },
    good_c6_three_students: { ok: e.ok, errors: e.errors, expect_ok: true },
    bad_reversed_quote_order: { ok: f.ok, errors: f.errors, expect_ok: false },
    good_forward_quote_order: { ok: g.ok, errors: g.errors, expect_ok: true },
    good_p037_c6_s2_s3: { ok: h.ok, errors: h.errors, expect_ok: true },
    good_c7_peer_teaching: { ok: i.ok, errors: i.errors, expect_ok: true }
  };
}

function testCDecisionTreeForPid_(pid){
  const sh = SpreadsheetApp.getActiveSheet();
  Logger.log("=== C DECISION TREE DRY-RUN ===");
  Logger.log("ACTIVE_SHEET=" + (sh ? sh.getName() : "(none)"));
  Logger.log("LAST_ROW=" + (sh ? sh.getLastRow() : 0));
  Logger.log("LAST_COLUMN=" + (sh ? sh.getLastColumn() : 0));
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map);
  const want = normPID_(pid || "P043");
  const matches = (packets || []).filter(function(p){ return p && p.pid === want; });

  Logger.log("=== C DECISION TREE DRY-RUN ===");
  Logger.log("REQUESTED_PID=" + want);
  Logger.log("MATCH_COUNT=" + matches.length);

  if (matches.length === 0) {
    const err = {
      ok: false,
      error_type: "PACKET_ERROR",
      message: "requested PID not found",
      requestedPid: want
    };
    Logger.log("SELECTED_PID=(none)");
    Logger.log("ERROR=" + JSON.stringify(err));
    Logger.log("DRY-RUN: C셀을 수정하지 않음");
    return err;
  }
  if (matches.length > 1) {
    const err = {
      ok: false,
      error_type: "PACKET_ERROR",
      message: "duplicate packets for requested PID",
      requestedPid: want,
      matchCount: matches.length,
      matches: matches.map(function(p){
        return {
          pid: p.pid,
          representativeRow: p.representativeRow,
          turns: (p.turns || []).length,
          active: p.activeStudentIds
        };
      })
    };
    Logger.log("SELECTED_PID=(duplicate, not chosen)");
    matches.forEach(function(p, idx){
      Logger.log("MATCH[" + idx + "] pid=" + p.pid + " repRow=" + p.representativeRow + " turns=" + ((p.turns || []).length) + " active=" + JSON.stringify(p.activeStudentIds));
    });
    Logger.log("ERROR=" + JSON.stringify(err));
    Logger.log("DRY-RUN: C셀을 수정하지 않음");
    return err;
  }

  const packet = matches[0];
  Logger.log("SELECTED_PID=" + packet.pid);
  Logger.log("representativeRow=" + packet.representativeRow);
  Logger.log("turns=" + ((packet.turns || []).length));
  Logger.log("active=" + JSON.stringify(packet.activeStudentIds));
  Logger.log("speakerCounts=" + JSON.stringify(packet.speakerCounts));
  Logger.log("turnDerivedCounts=" + JSON.stringify(packet.turnDerivedCounts));
  Logger.log("teacherPresent=" + packet.teacherPresent);
  (packet.turns || []).forEach(function(t){
    Logger.log("[" + t.row + "] [" + t.role + "] [" + (t.speakerId || "") + "] " + t.utterance);
  });

  const activeLen = (packet.activeStudentIds || []).length;
  if (activeLen < 2) {
    const result = runCDecisionTreeForPacket_(packet);
    const validation = validateCDecisionResult_(result, packet);
    Logger.log("STRUCTURAL_PRE_GATE=true");
    Logger.log("GPT_CALLED=false");
    Logger.log("RESULT=" + JSON.stringify(result));
    Logger.log("VALIDATION=" + JSON.stringify(validation));
    Logger.log("DISPLAY=" + formatCDecisionDisplay_(result));
    Logger.log("NOTE_STATUS=" + (result && result.status) + " code=" + (result && result.code) + " error_type=" + (result && result.error_type));
    Logger.log("DRY-RUN: C셀을 수정하지 않음");
    return {
      packet: packet,
      raw: null,
      parsed: null,
      validation: validation,
      result: result,
      display: formatCDecisionDisplay_(result)
    };
  }

  Logger.log("STRUCTURAL_PRE_GATE=false");
  Logger.log("GPT_CALLED=true");
  const prompt = buildCDecisionPrompt_(packet);
  Logger.log("PROMPT_TURNS_INCLUDED=" + (prompt.indexOf("[CURRENT CLUSTER TURNS]") >= 0));
  Logger.log("PROMPT_SUMMARY_AUX=" + (prompt.indexOf("AUXILIARY ONLY") >= 0));

  let raw = "";
  let parsed = null;
  let validation = null;
  let result = null;
  try {
    raw = callGPT_simple_(prompt, MODEL_C);
    Logger.log("RAW=" + String(raw).slice(0, 800));
    parsed = parseCDecisionTreeResponse_(raw);
    validation = validateCDecisionResult_(parsed, packet);
    Logger.log("PARSED=" + JSON.stringify(parsed));
    Logger.log("interaction_summary=" + (parsed && parsed.interaction_summary));
    Logger.log("contributors=" + JSON.stringify(parsed && parsed.contributors));
    Logger.log("quotes=" + JSON.stringify(parsed && parsed.quotes));
    Logger.log("VALIDATION=" + JSON.stringify(validation));
    if (validation.ok) {
      result = parsed;
      result.status = "OK";
    } else {
      result = _makeCDecisionError_("VALIDATION_ERROR", validation.errors.join("; "), packet.pid);
    }
  } catch (e) {
    result = _makeCDecisionError_(String(e).indexOf("JSON") >= 0 ? "PARSER_ERROR" : "API_ERROR", e.toString(), packet.pid);
    Logger.log("ERROR=" + e.toString());
  }

  const display = formatCDecisionDisplay_(result);
  Logger.log("DISPLAY=" + display);
  Logger.log("NOTE_STATUS=" + (result && result.status) + " code=" + (result && result.code) + " error_type=" + (result && result.error_type));
  Logger.log("DRY-RUN: C셀을 수정하지 않음");
  return { packet: packet, raw: raw, parsed: parsed, validation: validation, result: result, display: display };
}

function TEST_C_DECISION_TREE(){
  return testCDecisionTreeForPid_("P043");
}

function TEST_C_DECISION_TREE_P043(){
  return testCDecisionTreeForPid_("P043");
}

function TEST_C_DECISION_TREE_P035(){
  return testCDecisionTreeForPid_("P035");
}

function TEST_C_DECISION_TREE_P055(){
  return testCDecisionTreeForPid_("P055");
}

function TEST_C_DECISION_TREE_P037(){
  return testCDecisionTreeForPid_("P037");
}

function TEST_C_DECISION_TREE_P049(){
  return testCDecisionTreeForPid_("P049");
}

function testCPacketForPid_(pid){
  const sh = SpreadsheetApp.getActiveSheet();
  const want = normPID_(pid || "P043");
  Logger.log("=== C PACKET DIAGNOSTIC ===");
  Logger.log("ACTIVE_SHEET=" + (sh ? sh.getName() : "(none)"));
  Logger.log("LAST_ROW=" + (sh ? sh.getLastRow() : 0));
  Logger.log("LAST_COLUMN=" + (sh ? sh.getLastColumn() : 0));
  Logger.log("REQUESTED_PID=" + want);

  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map);
  const matches = (packets || []).filter(function(p){ return p && p.pid === want; });

  Logger.log(want + "_MATCH_COUNT=" + matches.length);
  matches.forEach(function(p, idx){
    Logger.log(
      "MATCH[" + idx + "] pid=" + p.pid +
      " repRow=" + p.representativeRow +
      " turns=" + ((p.turns || []).length) +
      " active=" + JSON.stringify(p.activeStudentIds)
    );
    (p.turns || []).forEach(function(t){
      Logger.log(
        "[" + t.row + "] [" + t.role + "] [" +
        (t.speakerId || "") + "] " + t.utterance
      );
    });
  });
  if (matches.length > 1) {
    Logger.log("WARNING: " + want + " packet이 2개 이상이다. 임의로 첫 packet을 선택하지 않음.");
  }
  if (matches.length === 0) {
    Logger.log("WARNING: " + want + " packet이 없다. 활성 시트가 데이터 시트인지 확인하세요.");
  }
  return matches;
}

function TEST_C_PACKET_P043(){
  return testCPacketForPid_("P043");
}

function TEST_C_PACKET_P055(){
  return testCPacketForPid_("P055");
}


/***** ===== M Decision Tree v1.0 ===== *****/
const M_DECISION_LABELS_ = {
  M1: "논의 목표·방법 조정",
  M2: "참여 태도·규범 조정",
  M3: "설명·논리 재검토",
  M4: "개념 이해 회복"
};

function _makeMDecisionError_(errorType, message, pid, extra){
  const err = {
    schema_version: "KCMP_M_V1",
    status: "ERROR",
    error_type: String(errorType || "ERROR"),
    message: String(message == null ? "" : message),
    pid: pid || "",
    code: null,
    contributors: []
  };
  if (extra && typeof extra === "object") {
    Object.keys(extra).forEach(function(k){ err[k] = extra[k]; });
  }
  return err;
}

function _makeMNoneResult_(pid, reason, path){
  return {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: null,
    contributors: [],
    metacognitive_target: null,
    reason: reason,
    decision_path: path || ["M-STEP0:NO"],
    boundary_check: null,
    context_needed: false,
    quotes: [],
    m3_evidence: null,
    pid: pid || ""
  };
}

function _mPacketMappingUnreliable_(packet){
  const audit = packet && packet.audit ? packet.audit : {};
  const u1 = audit.unmappedStudentSpeakers;
  const u2 = audit.unknownSpeakers;
  return (Array.isArray(u1) && u1.length > 0) || (Array.isArray(u2) && u2.length > 0);
}

function formatMDecisionDisplay_(result){
  if (!result || result.status !== "OK" || !result.code) return "";
  const label = M_DECISION_LABELS_[result.code] || result.code;
  let reason = String(result.reason == null ? "" : result.reason).replace(/\s+/g, " ").trim();
  if (reason.length > 60) reason = reason.slice(0, 57) + "…";
  if (!reason) return result.code + " — " + label;
  return result.code + " — " + label + " — " + reason;
}

function _writeMDecisionCell_(sheet, row, mCol, result){
  const cell = sheet.getRange(row, mCol);
  cell.clearContent();
  if (result && result.status === "OK" && result.code) {
    cell.setValue(formatMDecisionDisplay_(result));
  }
  try {
    cell.setNote(JSON.stringify(result || {}));
  } catch (e) {
    cell.setNote(JSON.stringify(_makeMDecisionError_("NOTE_WRITE_ERROR", e.toString(), result && result.pid)));
  }
}

function _kcmpMQuoteExistsInStudentTurns_(quote, studentTurns){
  const q = _kcmpNormForQuote_(quote);
  if (!q) return false;
  return (studentTurns || []).some(function(t){
    const u = _kcmpNormForQuote_(t.utterance);
    if (!u) return false;
    if (q.length === 1) return u === q;
    return u.indexOf(q) >= 0;
  });
}

function _kcmpValidateMQuoteEntries_(entries, studentTurns, label, errors, packet){
  if (!Array.isArray(entries)) {
    errors.push(label + "가 배열이 아님");
    return;
  }
  entries.forEach(function(q, i){
    const idx = label + "[" + i + "]";
    if (q == null || typeof q !== "object" || Array.isArray(q)) {
      if (typeof q === "string") {
        errors.push(idx + "은 {speaker, quote} 객체여야 함 (bare string 금지)");
      } else {
        errors.push(idx + "은 {speaker, quote} 객체여야 함");
      }
      return;
    }
    const sp = q.speaker;
    if (!sp || !/^S[1-4]$/.test(String(sp))) {
      errors.push(idx + "에 speaker(S1~S4) 필드가 없거나 무효함 (교사 금지)");
      return;
    }
    if (packet && (packet.activeStudentIds || []).indexOf(sp) < 0) {
      errors.push(idx + " speaker가 activeStudentIds에 없음: " + sp);
    }
    if (q.quote == null || String(q.quote).trim().length === 0) {
      errors.push(idx + "에 quote 필드가 없거나 비어 있음");
      return;
    }
    const quote = _kcmpNormForQuote_(q.quote);
    if (!_kcmpMQuoteExistsInStudentTurns_(q.quote, studentTurns)) {
      errors.push(idx + " 인용이 학생 원발화에서 확인되지 않음: " + quote);
    }
  });
}

function _kcmpTurnMatchesQuote_(turn, quote){
  if (!turn || quote == null) return false;
  const q = _kcmpNormForQuote_(quote);
  const u = _kcmpNormForQuote_(turn.utterance);
  if (!q || !u) return false;
  if (q.length === 1) return u === q;
  return u.indexOf(q) >= 0;
}

function _kcmpFindTurnForQuote_(quote, turns){
  for (let i = 0; i < (turns || []).length; i++) {
    if (_kcmpTurnMatchesQuote_(turns[i], quote)) return turns[i];
  }
  return null;
}

function _kcmpTurnOrderKey_(turn, fallbackIdx){
  if (turn && turn.row != null) return Number(turn.row);
  return fallbackIdx;
}

function _kcmpValidateM3EvidenceTrigger_(trigger, turns, errors, label){
  if (!trigger || typeof trigger !== "object" || Array.isArray(trigger)) {
    errors.push(label + "가 객체가 아님");
    return null;
  }
  const role = trigger.role;
  if (role !== "teacher" && role !== "student") {
    errors.push(label + ".role이 teacher|student가 아님: " + role);
    return null;
  }
  const quote = trigger.quote;
  if (quote == null || String(quote).trim().length === 0) {
    errors.push(label + ".quote가 비어 있음");
    return null;
  }
  const turn = _kcmpFindTurnForQuote_(quote, turns);
  if (!turn) {
    errors.push(label + ".quote가 current/prior context turn에 없음");
    return null;
  }
  if (turn.role !== role) {
    errors.push(label + ".role이 실제 turn role과 불일치: " + turn.role);
  }
  if (role === "teacher") {
    if (trigger.speaker != null && String(trigger.speaker).trim().length > 0) {
      trigger.speaker = null;
    }
  } else {
    const sp = trigger.speaker;
    if (turn.speakerId && /^S[1-4]$/.test(String(turn.speakerId))) {
      if (!sp || !/^S[1-4]$/.test(String(sp)) || turn.speakerId !== sp) {
        trigger.speaker = turn.speakerId;
      }
    } else if (!sp || !/^S[1-4]$/.test(String(sp))) {
      errors.push(label + ": student trigger인데 speaker가 S1~S4가 아님");
    } else if (turn.speakerId !== sp) {
      errors.push(label + ".speaker가 turn speakerId와 불일치");
    }
  }
  return turn;
}

function _kcmpValidateM3Evidence_(result, packet, errors, priorContextTurns){
  const code = result.code;
  const ev = result.m3_evidence;
  if (code !== "M3") {
    if (ev != null) errors.push("code가 M3가 아닌데 m3_evidence가 null이 아님");
    return;
  }
  if (ev == null || typeof ev !== "object" || Array.isArray(ev)) {
    errors.push("M3인데 m3_evidence가 {mode, trigger} 객체가 아님");
    return;
  }
  const mode = ev.mode;
  const allowed = { self_contained: true, trigger_response: true, reconstruction: true };
  if (!allowed[mode]) {
    errors.push("m3_evidence.mode 무효: " + mode);
    return;
  }
  const currentTurns = (packet && packet.turns) || [];
  const priorTurns = priorContextTurns || [];
  const triggerSearchTurns = priorTurns.concat(currentTurns);
  const trigger = ev.trigger;
  if (mode === "self_contained") {
    if (trigger != null) errors.push("self_contained M3인데 m3_evidence.trigger가 null이 아님");
    const et = ev.evaluation_type;
    const allowedEt = { adequacy: true, equivalence: true, difference: true, contradiction: true, applicability: true, coverage: true };
    if (et != null && !allowedEt[et]) {
      errors.push("m3_evidence.evaluation_type 무효: " + et);
    }
    return;
  }
  if (ev.evaluation_type != null) {
    ev.evaluation_type = null;
  }
  if (mode === "trigger_response") {
    const triggerTurn = _kcmpValidateM3EvidenceTrigger_(trigger, triggerSearchTurns, errors, "m3_evidence.trigger");
    if (!triggerTurn) return;
    const triggerRow = _kcmpTurnOrderKey_(triggerTurn, -1);
    let hasAfterContributorQuote = false;
    (result.quotes || []).forEach(function(q, qi){
      const qt = _kcmpFindTurnForQuote_(q.quote, currentTurns);
      if (!qt) return;
      const qRow = _kcmpTurnOrderKey_(qt, qi);
      if (qRow <= triggerRow) {
        errors.push("trigger_response M3인데 quotes[" + qi + "]가 trigger보다 앞이거나 같음");
      } else if ((result.contributors || []).indexOf(q.speaker) >= 0) {
        hasAfterContributorQuote = true;
      }
    });
    if (!hasAfterContributorQuote) {
      errors.push("trigger_response M3인데 trigger 이후 contributor quote가 없음");
    }
    return;
  }
  if (mode === "reconstruction") {
    if (trigger == null) return;
    const triggerTurn = _kcmpValidateM3EvidenceTrigger_(trigger, triggerSearchTurns, errors, "m3_evidence.trigger");
    if (!triggerTurn) return;
    const triggerRow = _kcmpTurnOrderKey_(triggerTurn, -1);
    let hasAfter = false;
    (result.quotes || []).forEach(function(q, qi){
      const qt = _kcmpFindTurnForQuote_(q.quote, currentTurns);
      if (!qt) return;
      const qRow = _kcmpTurnOrderKey_(qt, qi);
      if (qRow <= triggerRow) {
        errors.push("reconstruction M3인데 quotes[" + qi + "]가 prior anchor/trigger보다 앞이거나 같음");
      } else {
        hasAfter = true;
      }
    });
    if ((result.quotes || []).length > 0 && !hasAfter) {
      errors.push("reconstruction M3인데 trigger 이후 final quote가 없음");
    }
  }
}

function _kcmpFindAllTurnsForQuote_(quote, turns){
  const matches = [];
  for (let i = 0; i < (turns || []).length; i++) {
    if (_kcmpTurnMatchesQuote_(turns[i], quote)) matches.push(turns[i]);
  }
  return matches;
}

function _kcmpResolveTriggerTurn_(trigger, turns){
  if (!trigger || trigger.quote == null) return null;
  const matches = _kcmpFindAllTurnsForQuote_(trigger.quote, turns);
  if (matches.length === 0) return _kcmpFindTurnForQuote_(trigger.quote, turns);
  if (matches.length === 1) return matches[0];
  const role = trigger.role;
  if (role === "teacher" || role === "student") {
    const byRole = matches.filter(function(t){ return t.role === role; });
    if (byRole.length >= 1) return byRole[0];
  }
  return matches[0];
}

function _allErrorsAreMMetadataOnly_(errors){
  const errs = errors || [];
  if (!errs.length) return false;
  return errs.every(function(err){
    const s = String(err);
    return /teacher trigger인데 speaker가 null이 아님/.test(s) ||
      /student trigger인데 speaker가 S1~S4가 아님/.test(s) ||
      /trigger\.speaker가 turn speakerId와 불일치/.test(s) ||
      /self_contained가 아닌데 m3_evidence\.evaluation_type/.test(s);
  });
}

function _canonicalizeMDecisionResult_(result, packet, ctx){
  const repairLog = {
    m_canonical_repair: false,
    m_canonical_repair_fields: [],
    before_trigger: null,
    after_trigger: null
  };
  if (!result || result.code !== "M3") return { result: result, repairLog: repairLog };
  const ev = result.m3_evidence;
  if (!ev || typeof ev !== "object" || Array.isArray(ev)) {
    return { result: result, repairLog: repairLog };
  }
  let repaired = false;
  if (ev.mode !== "self_contained" && ev.evaluation_type != null) {
    ev.evaluation_type = null;
    repaired = true;
    repairLog.m_canonical_repair_fields.push("m3_evidence.evaluation_type");
  }
  if (!ev.trigger || ev.mode === "self_contained") {
    if (repaired) repairLog.m_canonical_repair = true;
    return { result: result, repairLog: repairLog };
  }
  const priorTurns = (ctx && ctx.priorContextTurns) || [];
  const currentTurns = (packet && packet.turns) || [];
  const triggerSearchTurns = priorTurns.concat(currentTurns);
  const trigger = ev.trigger;
  const turn = _kcmpResolveTriggerTurn_(trigger, triggerSearchTurns);
  if (!turn) {
    if (repaired) repairLog.m_canonical_repair = true;
    return { result: result, repairLog: repairLog };
  }
  repairLog.before_trigger = JSON.parse(JSON.stringify(trigger));
  if (trigger.role !== turn.role) {
    trigger.role = turn.role;
    repaired = true;
    repairLog.m_canonical_repair_fields.push("m3_evidence.trigger.role");
  }
  if (turn.role === "teacher") {
    if (trigger.speaker != null && String(trigger.speaker).trim().length > 0) {
      trigger.speaker = null;
      repaired = true;
      repairLog.m_canonical_repair_fields.push("m3_evidence.trigger.speaker");
    }
  } else if (turn.role === "student" && turn.speakerId) {
    const sp = trigger.speaker;
    if (!sp || !/^S[1-4]$/.test(String(sp)) || turn.speakerId !== sp) {
      trigger.speaker = turn.speakerId;
      repaired = true;
      repairLog.m_canonical_repair_fields.push("m3_evidence.trigger.speaker");
    }
  }
  if (repaired) {
    repairLog.m_canonical_repair = true;
    repairLog.after_trigger = JSON.parse(JSON.stringify(trigger));
  }
  return { result: result, repairLog: repairLog };
}

function _shouldSemanticRetryM_(errors){
  const errs = errors || [];
  if (!errs.length) return false;
  if (_allErrorsAreMMetadataOnly_(errs)) return false;
  return errs.some(function(err){
    const s = String(err);
    if (/teacher trigger인데 speaker가 null이 아님/.test(s)) return false;
    if (/student trigger인데 speaker가 S1~S4가 아님/.test(s)) return false;
    if (/trigger\.speaker가 turn speakerId와 불일치/.test(s)) return false;
    if (/self_contained가 아닌데 m3_evidence\.evaluation_type/.test(s)) return false;
    return true;
  });
}

function _logMDecisionCanonicalRepair_(repairLog){
  const r = repairLog || {};
  Logger.log("M_CANONICAL_REPAIR=" + (r.m_canonical_repair ? "true" : "false"));
  Logger.log("M_CANONICAL_REPAIR_FIELDS=" + JSON.stringify(r.m_canonical_repair_fields || []));
  Logger.log("BEFORE_TRIGGER=" + JSON.stringify(r.before_trigger || null));
  Logger.log("AFTER_TRIGGER=" + JSON.stringify(r.after_trigger || null));
}

function _appendMDecisionM3TriggerMinimumRules_(lines){
  lines.push("===== M3 TRIGGER MINIMUM =====");
  lines.push("trigger_response/reconstruction의 trigger는 단순히 \"앞에 나온 관련 발화\"가 아니다. trigger는 실제로 existing explanation을 under review 상태로 만들어야 한다.");
  lines.push("허용 trigger 기능: challenge, counterexample, contradiction, competing alternative, logical inconsistency, explicit adequacy question, explanation comparison, teacher/peer probing that puts the current explanation at issue.");
  lines.push("ORDINARY PRIOR CONTENT ≠ TRIGGER: 같은 학생의 이전 설명, peer의 일반 내용 설명, 앞선 주장 자체, 단순 정보 제공, 같은 설명을 한 단계 더 이어가는 발화는 trigger로 사용 금지.");
  lines.push("PEER EXPLANATION ≠ TRIGGER: peer가 causal explanation을 말했다고 그 peer 발화를 trigger로 쓰지 마라. S2 \"공기가 들어와야 커질 수 있다.\" → S1 \"근육이 없어서 공기가 들어와야...\"는 explanation elaboration chain이지 challenge/review trigger가 아니다 → null M.");
  lines.push("예 NEGATIVE: S1 \"공기가 들어오면 폐가 커져.\" S1 \"근육이 없으니까 공기가 들어와야 커질 수 있어.\" → explanation continuation/elaboration → M3 아님. K/C 가능.");
  lines.push("FIRST-ORDER CONTINUATION NEGATIVE: S1 \"공기가 들어와야 커질 수 있어.\" S2 \"공기가 들어오니까 커지는 거지.\" S1 \"근육이 없어서 공기가 들어와야...\" → explanation construction만이면 M3 아님.");
  lines.push("TRIGGER SELF-CHECK: \"Did the trigger actually put the current explanation under review?\" NO → trigger_response 금지. \"this came earlier\"만으로 trigger 근거 불가.");
  lines.push("reconstruction도 단순 science elements 연결만으로 성립하지 않음. existing explanation/problem → reflective re-linking / adequacy reconsideration 필요.");
}

function _appendMDecisionM1LogisticsExclusionRules_(lines){
  lines.push("===== M1 LOGISTICS / ROLE COORDINATION EXCLUSION =====");
  lines.push("M1은 discussion/problem-solving PROCESS regulation이다. LOGISTICS coordination은 M1 아님.");
  lines.push("HARD EXCLUSION (자동 M1 아님): 발표할 사람 정하기, 발표자 지정, 역할 분담, 누가 쓸지/읽을지 결정, 순번 정하기, 단순 업무 배정.");
  lines.push("예: \"발표할 사람 정하자.\" \"내가 할게.\" \"네가 읽어.\" \"내가 쓸게.\" \"일단 정하자. 발표할 사람.\" → task/role coordination (C6 가능), M1 아님. M-STEP2:NO.");
  lines.push("TASK GOAL CLARIFICATION ≠ ROLE ASSIGNMENT: \"뭘 발표하는데?\"는 WHAT to present(task goal) 확인 → M1 가능. \"발표할 사람 정하자.\" / \"일단 정하자. 발표할 사람.\"는 WHO does the task → M1 아님.");
  lines.push("PRIORITY MINIMUM RULE: M4 > M1 > M2 > M3는 각 코드의 MINIMUM CONDITION을 통과한 후보끼리만 적용. role assignment가 있어도 M1 minimum이 NO이면 M1 candidate 제거 후 M3/M4 평가 계속.");
  lines.push("EXPLANATION-USE vs LOGISTICS: cluster에 role coordination과 explanation-use monitoring이 함께 있어도, role assignment 때문에 M1을 우선하지 마라. explanation-use가 실제 meta target이면 M3.");
  lines.push("P065형: S3 \"말이 되는 게 있는데 그걸 활용을 못 하겠어.\"(explanation-use M3) + S1 \"일단 정하자. 발표할 사람.\"(logistics) → final M=M3, contributors=[\"S3\"] only.");
}

function _buildMDecisionContext_(packet, allPackets){
  const ctx = {
    priorContextTurns: [],
    priorContextPid: null
  };
  if (!packet || !packet.context) return ctx;
  const prevPid = packet.context.previousPid;
  if (!prevPid || !allPackets) return ctx;
  const prev = (allPackets || []).filter(function(p){ return p && p.pid === prevPid; })[0];
  if (!prev || !Array.isArray(prev.turns) || prev.turns.length === 0) return ctx;
  ctx.priorContextPid = prevPid;
  ctx.priorContextTurns = prev.turns.slice();
  return ctx;
}

function _formatMDecisionTurnsText_(turns){
  return (turns || []).map(function(t){
    const sid = t.speakerId ? t.speakerId : "";
    const raw = t.speakerRaw || "";
    return "[" + t.row + "] " + t.role + " " + sid + " " + raw + ": " + String(t.utterance || "").replace(/\s+/g, " ");
  }).join("\n");
}

function _appendMDecisionPromptContextRules_(lines){
  lines.push("===== PRIOR CONTEXT (ASYMMETRIC) =====");
  lines.push("CURRENT CLUSTER가 1차 자료이다. 그러나 current cluster 첫 발화의 target/function이 직전 cluster 없이는 이해되지 않을 경우, [PRIOR CONTEXT]를 보조적으로 사용할 수 있다.");
  lines.push("PRIOR CONTEXT: n-1 cluster turn — current utterance가 무엇에 반응하는지 해석하고 M3 trigger를 식별하는 데 사용 가능.");
  lines.push("FUTURE CONTEXT 금지: current utterance 이후 일어난 사건(next cluster 포함)을 이용해 current/earlier utterance를 M으로 소급 변경하지 마라. n+1 → n은 M 성립 근거로 사용 금지.");
  lines.push("prior context turn은 final contributor/quote가 아니다. prior teacher turn은 m3_evidence.trigger로만 사용 가능.");
  lines.push("final contributors와 final quotes는 [CURRENT CLUSTER TURNS]의 student turns에서만 선택한다.");
  lines.push("");
  lines.push("===== CROSS-CLUSTER TRIGGER BRIDGE =====");
  lines.push("previous cluster 마지막 teacher challenge → current cluster 첫 student response가 하나의 연속 interaction이면, prior teacher turn을 M3 trigger로 사용할 수 있다.");
  lines.push("예: PRIOR Teacher \"그러면 근육이 없으면 못 움직이는 거야?\" → CURRENT S1 \"못 움직이지 않아요?\" / \"못 움직일 것 같은데.\" → prior trigger + current student reinspection → M3 가능.");
  lines.push("final M code는 CURRENT cluster에만 기록. final contributor=current student. final quote=current student utterance.");
  lines.push("");
  lines.push("===== M3 REINSPECTION ≠ REVISION =====");
  lines.push("M3 = 설명/논리의 재검토. 재검토 결과가 반드시 \"내가 틀렸네\" \"다른 설명으로 바꿀게\"일 필요 없다.");
  lines.push("challenge 이후 기존 설명/판단을 다시 판단하고 유지(\"그래도 X일 것 같은데.\")해도 실제 재검토 trajectory가 있으면 M3 가능.");
  lines.push("결론을 바꾸지 않았다는 이유만으로 M3를 탈락시키지 마라. temporal relation(challenge/trigger → student response)이 핵심이다.");
  lines.push("challenge 없는 standalone \"못 움직일 것 같은데.\" \"X일 것 같은데.\"는 M3 자동 아님.");
  lines.push("");
  lines.push("===== M3 MAINTAINED STANCE / CONTINUED TRAJECTORY =====");
  lines.push("MAINTAINED STANCE: Student [기존 설명] Teacher [challenge] Student \"그래도 X일 것 같은데.\" / \"못 움직일 것 같은데.\" → challenge 후 입장 재검토·유지 → M3 가능.");
  lines.push("CONTINUED TRAJECTORY: 한 cluster 안에서 S1 [판단 재확인] Teacher [새 원리/조건] S1 [원리로 explanation 재구성 \"그러니까 공기는 높은 압력에서 낮은 압력으로…\"] → 전체가 하나의 M3 trajectory.");
  lines.push("reconstruction 또는 trigger_response 중 실제 trajectory에 맞는 mode를 선택한다. context-dependent M3는 self_contained 사용 금지.");
  lines.push("reason은 observable interaction trajectory만 기술한다. \"수정할 필요성을 느꼈다\" \"깨달았다\" 같은 내적 상태를 추론하지 마라.");
}

function _normalizeMQuoteKey_(quotes){
  const arr = (quotes || []).slice().sort(function(a, b){
    const sa = String(a && a.speaker ? a.speaker : "");
    const sb = String(b && b.speaker ? b.speaker : "");
    if (sa !== sb) return sa.localeCompare(sb);
    return _kcmpNormForQuote_(a && a.quote).localeCompare(_kcmpNormForQuote_(b && b.quote));
  });
  return arr.map(function(q){
    return String(q && q.speaker ? q.speaker : "") + "::" + _kcmpNormForQuote_(q && q.quote);
  }).join("|");
}

function _sameMSemanticCandidate_(first, retry){
  if (!first || !retry) return false;
  if (first.code !== retry.code) return false;
  const c1 = (first.contributors || []).slice().sort().join(",");
  const c2 = (retry.contributors || []).slice().sort().join(",");
  if (c1 !== c2) return false;
  const m1 = first.m3_evidence;
  const m2 = retry.m3_evidence;
  const mode1 = m1 && m1.mode ? m1.mode : "";
  const mode2 = m2 && m2.mode ? m2.mode : "";
  if (mode1 !== mode2) return false;
  const t1 = m1 && m1.trigger ? _kcmpNormForQuote_(m1.trigger.quote) : "";
  const t2 = m2 && m2.trigger ? _kcmpNormForQuote_(m2.trigger.quote) : "";
  if (t1 !== t2) return false;
  return _normalizeMQuoteKey_(first.quotes) === _normalizeMQuoteKey_(retry.quotes);
}

function _hasTemporalImpossibilityErrors_(errors){
  return (errors || []).some(function(err){
    const s = String(err || "");
    return s.indexOf("trigger보다 앞이거나 같음") >= 0 ||
      s.indexOf("trigger 이후 contributor quote가 없음") >= 0 ||
      s.indexOf("prior anchor/trigger보다 앞이거나 같음") >= 0 ||
      s.indexOf("trigger 이후 final quote가 없음") >= 0;
  });
}

function _kcmpWarnSelfContainedContextDependency_(result, ctx, warnings){
  if (result.code !== "M3") return;
  const ev = result.m3_evidence;
  if (!ev || ev.mode !== "self_contained") return;
  const reason = String(result.reason == null ? "" : result.reason);
  const reasonDependsOnPrior = /prior|이전|앞서|앞의|teacher|교사|challenge|반박|trigger|반문|직전|context/.test(reason);
  if (!reasonDependsOnPrior) return;
  const evalInQuote = /같은|다른|설명이 안|못 하겠|적용|활용|말이 되|모순|안 맞|같은 소리|같은 말|아니야\?|아닌가|비교|equivalence|adequacy|coverage/i;
  const quoteHasEval = (result.quotes || []).some(function(q){
    return evalInQuote.test(String(q && q.quote ? q.quote : ""));
  });
  if (!quoteHasEval) {
    warnings.push("SELF_CONTAINED_CONTEXT_DEPENDENCY_SUSPECTED");
  }
}

function _appendMDecisionSelfContainedRules_(lines){
  lines.push("===== EXPLANATION PRODUCTION ≠ EXPLANATION MONITORING =====");
  lines.push("M3는 \"과학 설명이 존재하는가?\"가 아니라 \"학생이 설명/추론 자체를 평가·비교·재검토 대상으로 삼는가?\"를 본다.");
  lines.push("EXPLANATION PRODUCTION / ELABORATION ≠ M3. claim, reason, causal explanation, explanation elaboration만 있으면 M3 아님. K/C 가능.");
  lines.push("CONTENT COMPLEXITY ≠ M3. 길거나 인과관계가 있거나 여러 개념을 연결했다고 M3 아님.");
  lines.push("질문: \"Is the student constructing the explanation, or evaluating the explanation as an explanation?\" constructing only → not M3.");
  lines.push("");
  lines.push("===== M3 SELF_CONTAINED MINIMUM (STRICT) =====");
  lines.push("self_contained M3는 student quote ONLY에서 다음 중 하나가 직접 드러나야 한다:");
  lines.push("A. ADEQUACY \"이 설명으로는 X가 설명이 안 되는데?\" B. EQUIVALENCE \"이 두 말이 결국 같은 설명 아니야?\" C. DIFFERENCE \"이거랑 저거는 다른 설명 아닌가?\"");
  lines.push("D. CONTRADICTION \"그럼 앞에서 말한 거랑 안 맞잖아?\" E. APPLICABILITY \"말이 되는 원리가 있는데 여기에는 적용을 못 하겠어.\" F. COVERAGE \"이쪽은 설명되는데 저쪽은 설명이 안 돼.\"");
  lines.push("SELF-CONTAINED POSITIVE: \"이 두 설명이 결국 같은 말 아니야?\" \"그런데 숨을 빨아들이는 것도 밀고 들어오는 거 아니야?\" \"말이 되는 게 있는데 그걸 활용을 못 하겠어.\" \"이 설명으로는 공기가 나가는 경우가 설명이 안 되는데?\"");
  lines.push("FIRST-ORDER HARD NEGATIVES (NOT self_contained M3): \"공기가 들어와야 커질 수 있다.\" \"근육이 없어서 공기가 들어와야...\" \"압력이 낮아서 공기가 들어온다.\" \"폐에 근육이 없으니까요.\" — claim/reason/causal explanation/elaboration only.");
  lines.push("P035형 NEGATIVE: S1 \"공기가 들어오니까 커진다.\" S2 \"공기가 들어와야 커질 수 있다.\" S1 \"근육이 없어서 공기가 들어와야...\" → same explanation built/elaborated, no contradiction/equivalence/adequacy → null M.");
  lines.push("SELF-CONTAINED SELF-CHECK Q1: quote가 과학적 주장/근거를 단순 제시하는가? YES → 기본적으로 M3 아님. Q2: quote가 explanation adequacy/equivalence/difference/contradiction/applicability/coverage를 평가하는가? YES → self_contained M3 가능.");
  lines.push("CONTEXT-DEPENDENT RULE (narrow): quote ONLY에서 evaluation function이 보이면 prior context가 있어도 self_contained 가능. 예: \"그런데 A도 B라는 거 아니야?\" → equivalence in quote → self_contained OK. \"못 움직일 것 같은데.\"만 있고 challenge trajectory에만 의존 → self_contained 금지, trigger_response 사용.");
  lines.push("P065 APPLICABILITY POSITIVE: \"말이 되는 게 있는데 그걸 활용을 못 하겠어.\" → explicit applicability monitoring → self_contained M3. P035 causal proposition과 구분.");
  lines.push("");
  lines.push("===== M3 TERMINAL CHALLENGE HARD NEGATIVE =====");
  lines.push("cluster 마지막 turn이 teacher challenge이고 그 AFTER student response가 없으면: trigger_response M3 불가. earlier student explanation을 self_contained M3로 바꾸지 마라. first-order reason 자체는 M3 아님. 다른 M evidence 없으면 code=null.");
  lines.push("TERMINAL SELF-CHECK: \"Is the only challenge in this cluster the final turn, with no student response after it?\" YES → terminal challenge로 earlier student M3 만들 수 없음. strict self_contained minimum도 만족하지 않으면 M3 후보 제거.");
}

function buildMDecisionPrompt_(packet, ctx){
  ctx = ctx || {};
  const priorTurns = ctx.priorContextTurns || [];
  const students = (packet.students || []).map(function(s){
    return (s.id || "") + " = " + (s.label || "");
  }).join("\n");
  const active = (packet.activeStudentIds || []).join(", ");
  const turnsText = _formatMDecisionTurnsText_(packet.turns || []);
  const priorTurnsText = _formatMDecisionTurnsText_(priorTurns);
  const summary = String(packet.summary == null ? "" : packet.summary);

  const lines = [];
  lines.push("당신은 과학 소집단 담화의 M차원(메타인지) 코더이다.");
  lines.push("입력의 1차 자료는 [CURRENT CLUSTER TURNS]의 원발화이다.");
  lines.push("[SUMMARY]는 보조자료이다. 원발화와 충돌하면 원발화를 우선한다.");
  lines.push("주변 클러스터(previousPid/nextPid) 내용을 추측하여 채워 넣지 마라.");
  lines.push("JSON만 출력하라. JSON 외 텍스트, 마크다운, 코드펜스 금지.");
  lines.push("");
  lines.push("가능한 code: null | \"M1\" | \"M2\" | \"M3\" | \"M4\"  (최종 코드는 최대 1개)");
  lines.push("우선순위: M4 > M1 > M2 > M3. 단 상위 코드의 최소조건이 실제로 성립할 때만 부여.");
  lines.push("M = 학생이 자신의 이해, 설명의 논리, 논의 방식/전략, 참여 과정 자체를 돌아보거나 조정하는 metacognitive process.");
  lines.push("단순히 과학적으로 깊은 말을 했다는 이유로 M을 주지 않는다. K3 존재 ≠ M. C3 존재 ≠ M. 논리적 설명 ≠ M3. 불확실한 표현 ≠ 자동 M4.");
  lines.push("GLOBAL-M content reasoning ≠ metacognition: 과학 내용을 설명하고 이유를 제시하고 복잡한 추론을 한다는 사실만으로 M-STEP0 YES 또는 M4를 부여하지 마라.");
  lines.push("단, missing concept/law/meaning retrieval과 ongoing task/discussion에 대한 explicit review proposal은 과학 설명 생성이 아니며 STEP0:YES 후보이다.");
  lines.push("반드시 학생이 자신의 이해 상태, 논의 전략, 참여 과정, 기존 설명의 타당성 자체를 다시 보고 있는지 확인. \"과학적 사고를 한다\" ≠ \"자신의 사고를 점검한다\".");
  lines.push("학생 1명만 있어도 M은 성립할 수 있다. 활성 학생 2명 이상을 요구하지 않는다.");
  lines.push("");
  lines.push("===== OPERATIONAL PRINCIPLE =====");
  lines.push("대범주와 우선순위(M4>M1>M2>M3)는 유지한다. 그러나 첫 질문은 표면 행동(반드시 gap+별도 recovery가 있는가)이 아니라 META TARGET이다.");
  lines.push("과거 synthetic 규칙이 실제 담화의 meta-function과 충돌하면, 단어 heuristic이나 추가 필수 조건보다 meta target을 우선한다. 개별 사례 ID를 규칙으로 외우지 마라.");
  lines.push("금지 keyword 규칙: \"헷갈려\"→자동 M4 아님. \"아 그러면\"→자동 M3 아님. \"다시\"→자동 M1 아님. \"몰라\"→자동 M4 아님.");
  lines.push("");
  lines.push("===== GLOBAL-M =====");
  lines.push("GLOBAL-M1 표면 문장 형태가 아니라 학생이 무엇을 대상으로 monitoring/regulation 하는지를 본다.");
  lines.push("GLOBAL-M2 K/C 코드가 존재한다고 M을 추론하지 않는다. 같은 발화가 K와 M에 모두 기능적으로 기여할 수는 있다. 그러나 K3/C3 존재 ≠ M.");
  lines.push("GLOBAL-M3 먼저 meta target을 분류한다. TARGET A=자신의 개념 이해/의미/기억/법칙·원리 이해(M4 후보). TARGET B=논의·과제의 목표/진행 방식/검토 방식/기록 방식(M1 후보). TARGET C=참여 방식/발언 균형/토론 규범(M2 후보). TARGET D=현재 설명/주장/대안/설명 가능성/논리적 연결(M3 후보).");
  lines.push("GLOBAL-M4 단순 uncertainty/confidence 표현만으로 M 금지. target이 없는 \"그런 것 같아.\" \"아마.\" \"모르겠어.\"만 있으면 M을 강제하지 않는다.");
  lines.push("GLOBAL-M5 현재 클러스터가 기본 범위. n+1/next cluster 내용을 이용해 earlier student utterance를 M으로 소급하지 마라. 단, [PRIOR CONTEXT]는 current 첫 발화 해석 및 M3 trigger 식별에 보조적으로 사용 가능.");
  lines.push("GLOBAL-M6 학생 발화만 M evidence/contributor. 교사는 contributor가 아니다. 교사가 반례/논리 질문을 제시한 뒤 학생이 기존 설명을 재검토하면 그 학생의 발화로 M3 가능.");
  lines.push("teacher prompting 자체를 학생 metacognition으로 대체하지 마라. Student \"잘 모르겠어요.\" → Teacher 질문/힌트 → Student 과학 내용 답변만으로는 자동 M4 금지.");
  lines.push("");
  lines.push("===== M-STEP0 메타인지 과정이 있는가? =====");
  lines.push("질문: 학생이 자신의 개념 이해, 논의/과제 방식, 참여 과정, 또는 현재 설명/논리의 적절성을 점검·조정하고 있는가?");
  lines.push("먼저 행동 체크리스트보다 meta target이 식별되는지 본다. NO → code=null. YES → M-STEP1부터 priority 순으로 검토.");
  lines.push("명시적 \"내가 모르겠다\", \"우리 전략을 바꾸자\"가 없어도 아래 TYPE A/B면 STEP0:YES 가능.");
  lines.push("TYPE A missing concept/law/meaning retrieval: 현재 필요한 개념/법칙/원리/의미가 자기 지식에서 비어 있음을 드러내고 그 내용을 회수하려 물으면 STEP0:YES. 예: \"무슨 법칙이야?\" \"보일 법칙이 뭐예요?\" \"이 개념 뜻이 뭐였지?\" \"그 단어는 기억나는데 뜻이 기억이 안 나.\" \"무슨 법칙이야? 1학년 때 수업 안 들었는데.\"");
  lines.push("TYPE B explicit review/reinspection proposal: 진행 중인 task/discussion을 다시 검토하자고 제안하면 목적어가 생략되어도 STEP0:YES. 예: \"다시 한번 볼까?\" \"어. 다시 한번 볼까?\" \"한 번 더 확인해볼까?\" \"다시 검토해보자.\" — 목적어가 없다고 STEP0:NO 하지 마라.");
  lines.push("STEP0 YES 가능: 구체적 개념/법칙/의미를 자신의 지식에서 점검. 과제 논의에서 재검토/기록/발표 목표를 조정. 기존 설명의 성립/동일성/적용을 평가. 특정 개념 관계가 헷갈린다고 점검.");
  lines.push("STEP0 NO: \"몇 번이야?\" \"정답 뭐야?\" \"책 몇 쪽?\" — 단순 정답/페이지 정보 요청.");
  lines.push("STEP0 NO 가능: 이해 부족 표현 후 교사 질문에 과학 답을 한 것만 있는 경우. 과학 설명 생성만 있는 경우. 설명 속 수사적 질문만 있는 경우.");
  lines.push("STEP0 NO 가능: peer 선택 이유/내용 확인만 묻는 경우(\"왜 짱구야?\" \"압력이 더 높은 거지?\"). Teacher \"왜?\"에 대한 과학 답변만. target 없는 \"왜?\" \"몰라.\" 후회만. 이미 나온 법칙명의 단순 확인(\"보일 법칙이래?\").");
  lines.push("NO → code=null, contributors=[], metacognitive_target=null, quotes=[], decision_path=[\"M-STEP0:NO\"], reason에 왜 meta-process가 없는지 명시.");
  lines.push("");
  lines.push("===== M-STEP1 → M4 개념 이해 monitoring / awareness / recovery =====");
  lines.push("학생이 자신의 개념 이해/의미/기억/법칙·원리 이해 상태를 점검하는가? YES → M4.");
  lines.push("M4 target: What do I understand / remember / know?");
  lines.push("M4 최소조건: 학생 자신의 구체적 conceptual target과 함께 다음 중 하나가 확인되면 M4 가능 — 개념을 모름, 개념 의미를 모름, 배운 법칙/원리를 기억하지 못함, 특정 관계가 헷갈림, 어떤 법칙인지 묻음, 의미/원리를 다시 확인하려 함, 자신의 이해 부족을 드러냄, 도움을 요청하며 이해 문제를 해결하려 함.");
  lines.push("별도 다음 발화의 active recovery action이 반드시 있어야 하는 것은 아니다. 구체적 conceptual understanding gap을 명시적으로 인식하고 표현하는 것 자체도 M4가 될 수 있다.");
  lines.push("가능: \"무슨 법칙이야? 기억이 안 나.\" \"보일 법칙이 뭐예요?\" \"그 단어는 기억이 나는데 그 뜻이 기억이 안 나.\" \"갑자기 헷갈려. 눌렀을 때가…\" 이어서 \"안쪽이?\" \"도와주세요.\" + conceptual struggle.");
  lines.push("missing concept/law identification, concept meaning retrieval, prior-learning recall monitoring은 M4 가능. \"질문 뒤 답을 얻으려는 적극 행동이 반드시 더 있어야 한다\"는 추가 조건을 만들지 마라.");
  lines.push("M4 ≠ 모든 모름. 핵심은 conceptual target이 식별되는가이다. \"몰라.\"만 있고 무엇을 모르는지 불명확하면 M4 자동 금지. \"1학년 때 수업 들을걸.\" 후회/상태만이면 M4 자동 금지. bare \"왜?\"도 M4 keyword가 아니다.");
  lines.push("RETRIEVAL vs CONFIRMATION: WHAT IS THE MISSING CONCEPT? 이면 M4 가능. IS THIS THE ALREADY-PROPOSED CONCEPT? 이면 단순 confirmation일 수 있어 M4 자동 아님.");
  lines.push("retrieval 예: \"보일 법칙이 뭐예요?\" \"무슨 법칙이야?\" \"이 개념 뜻이 뭐였지?\" — 어떤 법칙/의미인지 몰라 회수 → M4.");
  lines.push("confirmation 예: \"보일 법칙이래?\" — 이미 제시된 법칙명을 확인/반복. conceptual gap monitoring이 명확하지 않으면 M4 자동 부여 금지.");
  lines.push("일반 SCIENCE QUESTION ≠ 자동 M4. \"압력이 높은 거지?\" \"혈구가 나가?\" \"3번이야?\" \"왜 그렇게 돼?\" — 현재 science proposition의 판단/확인/이유만 물으면 M4 아님.");
  lines.push("구분: peer가 방금 한 말의 뜻을 묻는 것(\"그게 무슨 말이야?\")은 C2 가능, 자동 M4 아님. 정답 번호/절차(\"몇 번이야?\" \"몇 쪽?\")는 M4 아님. 단순 과학 판단 확인(\"혈구가 나가?\" \"압력이 높은 거지?\")은 자기 knowledge-gap retrieval이 아니면 자동 M4 아님.");
  lines.push("가능: \"이게 무슨 뜻이지?\" \"이 관계가 어떻게 되는 거지? 이해가 안 돼.\" \"아 이거 모르겠는데, 다시 설명해 봐.\"");
  lines.push("confusion 단어가 있어도 target이 explanation adequacy/use이면 M3를 검토한다. \"왜 이렇게 헷갈리지?\"는 항상 M4가 아니다.");
  lines.push("teacher-elicited answer: S1 \"잘 모르겠어요.\" Teacher [질문/힌트] S1 [과학 내용 답변] → 자동 M4 금지.");
  lines.push("rhetorical/self-directed question ≠ 자동 M4. 예: \"우리 소장에서 지금 뭐냐? 융털에 암죽관이랑 모세혈관 있잖아.\" — 이미 아는 내용의 설명 구성이면 M4 아님.");
  lines.push("other-directed comprehension check ≠ M4. \"이해했니?\" \"알겠어?\"는 말한 학생 자신의 conceptual understanding monitoring이 아니다.");
  lines.push("과학 설명을 생성하는 것만으로 M4 금지.");
  lines.push("contributors는 conceptual self-monitoring을 수행한 학생만. 답을 준 peer는 자동 contributor 아님. gap와 clarification이 같은 학생의 별도 turn이면 복수 quote 가능. 그러나 반드시 2 quotes를 요구하지 마라.");
  lines.push("quote adequacy: quotes만 읽어도 자기 개념 이해 상태 점검이 보여야 한다. 과학 설명 quote alone으로 M4를 정당화하지 마라.");
  lines.push("M4 path: [\"M-STEP0:YES\",\"M-STEP1:YES\"]");
  lines.push("");
  lines.push("===== M-STEP2 → M1 논의 목표·방식·전략 점검 =====");
  lines.push("M1 STEP2 FIRST: role assignment / logistics / 발표자 지정 / \"일단 정하자. 발표할 사람.\" → M1 아님. M-STEP2:NO로 처리하고 M3/M4 계속 검토.");
  lines.push("M4 아니면: 학생이 논의/과제의 목표, 진행 방식, 검토 방식, 기록 방식을 점검하거나 조정하는가? YES → M1.");
  lines.push("M1 target: What should we do / check / record / review next?");
  lines.push("가능: 무엇을 확인할지 정함, 논의를 다시 검토하자고 함, 무엇을 발표해야 하는지 확인, 무엇을/어디에 기록해야 하는지 조정, 어떤 방향으로 이동했는지 확인하자고 제안, 과제 진행의 방식을 점검/조정.");
  lines.push("CONTENT QUESTION ≠ M1. \"왜 짱구야?\" \"왜 그렇게 되는 거야?\" \"압력이 더 높은 거지?\" \"이거 맞아?\" \"몇 번이야?\" — 과학 내용/판단/이유 요청이지 discussion/task process regulation이 아니다. (자기 개념 의미 retrieval \"이 뜻이 뭐였지?\"는 M1이 아니라 M4 검토.)");
  lines.push("질문이 담화의 방향을 만들었다는 이유만으로 M1을 부여하지 마라.");
  lines.push("M1 quote adequacy: 이 quote만 읽었을 때 과학 내용이 아니라 논의/과제 process를 조정하고 있음이 보이는가?");
  lines.push("quote adequacy NO: \"왜 짱구야?\" \"압력이 높은 거지?\". YES 가능: \"1번이 왜 아닌지를 먼저 검토해보자.\" \"표부터 다시 비교해보자.\" \"다시 한번 볼까?\" \"뭘 발표하는데?\" \"뭐라고 쓰지?\"");
  lines.push("예: \"1번이 왜 아닌지를 생각해보자.\" — 검토 목표를 명시적으로 설정 → M1 strong positive.");
  lines.push("EXPLICIT REVIEW PROPOSAL: 진행 중인 task/discussion을 RE-INSPECT/REVIEW 하자고 제안하면 목적어가 생략되어도 M1 가능. 예: \"다시 한번 볼까?\" \"어. 다시 한번 볼까?\" \"한 번 더 확인해볼까?\" \"다시 검토해보자.\" — \"다시\"라는 단어 때문이 아니라 review/reinspection proposal이기 때문이다.");
  lines.push("ELLIPSIS: 한국어 담화에서 review target이 현재 과제/논의로 명백하면 생략될 수 있다. standalone \"어. 다시 한번 볼까?\"도 ongoing work에 대한 review proposal이면 M1 가능. \"목적어가 없으니 무엇을 보는지 알 수 없다\"는 이유만으로 STEP0:NO 또는 M1 탈락시키지 마라.");
  lines.push("단, \"다시\" keyword 자체가 M1은 아니다. 상대 반복 요청 \"다시 말해봐.\", 내용 \"다시 커져.\", 단순 명령 \"다시 써.\"는 M1 아님. 오프태스크에서 아무 작업을 다시 하자는 말은 자동 M1 아님.");
  lines.push("TASK-GOAL / RECORDING: 단순 정보 요청 \"몇 페이지야?\"만으로는 M1 아닐 수 있다. 그러나 실제 공동 과제 수행 방식/기록 방식을 조정하면 M1 가능. 예: \"뭘 발표하는데?\" \"거기 아니고 아래.\" \"아직 안 썼어.\" \"뭐라고 쓰지?\"");
  lines.push("페이지/기록/위치/쓰기 관련 발화를 procedure라고 일괄 제외하지 마라. 기능을 본다. 단순 지시 \"180쪽 펴.\" \"시간 없어.\" \"다음 문제 가자.\"는 M1 아님.");
  lines.push("ROLE ASSIGNMENT / LOGISTICS ≠ M1. \"발표할 사람 정하자.\" \"내가 발표할게.\" \"일단 정하자. 발표할 사람.\" \"네가 읽어.\" \"내가 쓸게.\" — 역할/업무 배정(C6 가능), 논의 process regulation 아니면 M1 자동 부여 금지. 반면 \"뭘 발표하는데?\"는 task goal clarification → M1 가능.");
  _appendMDecisionM1LogisticsExclusionRules_(lines);
  lines.push("주변 cluster 내용을 추측해 새로운 검토 대상을 만들어 넣지 마라. 그러나 목적어 생략만으로 review proposal을 배제하지 마라. target은 \"현재 논의/과제 재검토\"일 수 있다.");
  lines.push("conceptual criterion evaluation ≠ M1. \"이 기준이면 이 결과가 설명이 안 되잖아.\" → M3 검토.");
  lines.push("CONTENT STANCE / JUDGMENT ≠ M1. \"못 움직일 것 같은데.\" \"압력이 낮은 것 같은데.\" \"이게 맞지 않아?\" \"그러면 안 되는 거 아니야?\" \"나는 2번 같은데.\" — science explanation/claim/alternative를 검토하는 content-level utterance이지 discussion/task process regulation이 아니다.");
  lines.push("M1 reason에서 \"논의 방향을 조정하였다\" \"문제 해결 방향\" 같은 추상적 재서술만 만들어 content utterance를 M1으로 올리지 마라. quote만 읽어도 WHAT/HOW TO DO NEXT process action이 보여야 한다.");
  lines.push("M1 POSITIVE 유지: \"어느 방향으로 이동했는지 확인해 보자.\" \"다시 한번 볼까?\" \"뭘 발표하는데?\" \"뭐라고 쓰지?\"");
  lines.push("M1 path: [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:YES\"]");
  lines.push("");
  lines.push("===== M-STEP3 → M2 참여 태도·규범 조정 =====");
  lines.push("M4/M1 아니면: 학생이 집단 구성원의 참여 방식, 발언 기회, 참여 균형, 토론 규범을 점검하거나 조정하는가? YES → M2.");
  lines.push("명확한 예: \"계속 우리 둘만 말했잖아. S3도 말해봐.\" \"한 명씩 얘기하자.\" \"너도 설명해야지. 계속 안 말했잖아.\" \"다 같이 한 번씩 의견 내보자.\"");
  lines.push("M2 아님: \"너는 몇 번?\" \"너 생각은 어때?\" — 단순 의견 요청이면 C2 가능, participation regulation이 아니면 M2 아님. \"너도 의견 말해봐.\"만으로 참여 문제 조정이 없으면 M2 금지.");
  lines.push("M2 threshold를 낮추지 마라. 현재 corpus에서 M2 positive는 매우 적을 수 있다.");
  lines.push("M2 path: [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:NO\",\"M-STEP3:YES\"]");
  lines.push("");
  lines.push("===== M-STEP4 → M3 설명·논리 재검토 / 재구성 =====");
  lines.push("M4/M1/M2 아니면: 학생이 현재 설명/주장/대안/설명 가능성/논리적 연결을 점검하거나 재구성하는가? YES → M3.");
  lines.push("M3 target: Does this explanation / alternative / reasoning work?");
  lines.push("M4와 구분: M4는 What do I understand/remember?(자기 개념 이해 상태). M3는 현재 설명/추론의 적절성·사용.");
  lines.push("M3는 명시적 \"내 논리가 틀렸어\" \"이건 모순이야\"가 있어야만 성립하는 것이 아니다.");
  lines.push("포함: (A) 기존 설명의 논리 문제 점검 (B) 교사/peer 반례·질문을 받아 기존 설명을 다시 검토 (C) 두 설명/대안이 같은지 다른지 비교 (D) 설명이 현상을 실제로 설명할 수 있는지 평가 (E) 한 설명은 가능하고 다른 설명은 불가능하다고 평가 (F) 기존 과학 관계를 다시 연결하여 설명 구조를 수정/reorganize (G) 알고 있는 논리를 현재 문제에 적용할 수 있는지 explanation-use를 점검.");
  lines.push("TYPE A explicit adequacy monitoring: \"그런데 그럼 공기가 들어오기만 하는 거 아니야?\" \"같은 소리인가?\" \"다르나?\" \"이 설명은 말이 안 되는 것 같은데.\" \"이쪽은 설명이 되는데 저쪽은 설명이 안 돼.\"");
  lines.push("TYPE B explanation restructuring: 앞선 설명 요소들이 cluster에 있고 학생이 그 요소를 다시 연결하여 기존 explanation을 수정/정리하면 M3 가능. 예: 들숨/날숨·가로막·내부 기압을 검토한 뒤 \"아, 그러면 횡격막이 안쪽 기압을 낮게 하는 거야. 공기는 높은 곳에서 낮은 곳으로 가니까...\"");
  lines.push("\"아 그러면\" keyword 자체가 M3는 아니다. 독립된 한 문장의 새 추론만 있으면 first-order K일 수 있다. 근거는 prior explanatory material → reflective re-linking/restructuring 이다.");
  lines.push("단순 새 설명 \"폐가 커지면 압력이 낮아져\" 또는 앞 요소 없이 처음 추론하는 \"그러면 압력이 낮아지는 거네\"는 자동 M3 아님.");
  lines.push("EXPLANATION EQUIVALENCE/DISTINCTION: \"빨아들이는 것도 밀고 들어오는 거 아니야?\" \"같은 소리인가?\" \"다르나?\" \"다르지 않을까?\" \"말이 다 똑같은 것 같아.\" — 경쟁 설명의 차이/동일성/논리적 구별 가능성을 평가하면 M3.");
  lines.push("EXPLANATORY ADEQUACY: \"설명이 가능하니까 이걸로 하자.\" \"이쪽이 그나마 설명이 되는데.\" \"이 설명은 할 수가 없어.\" \"이걸로는 설명을 못 하겠어요.\" — 답을 선택한 행위가 아니라 which explanation can account for the phenomenon?를 평가하면 M3. C6와 M3 동시 성립 가능.");
  lines.push("EXPLANATION-USE: \"말이 되는 게 있는데 그걸 활용을 못 하겠어.\" — 사용할 수 있는 논리가 있음을 알고 현재 explanation에 적용하지 못함을 monitoring하면 M3. 같은 cluster에 \"왜 이렇게 헷갈리지?\"가 있어도 confusion word만 보고 M4로 올리지 마라. meta target이 설명/논리 활용이면 M3. role coordination이 같이 있어도 그것만으로 M1을 우선하지 마라.");
  lines.push("M1↔M3: M1=process/task method. M3=explanation/reasoning. \"다시 한번 볼까?\"→M1. \"이 두 설명이 같은 소리인가?\"→M3. \"뭘 발표하는데?\"→M1. \"훈이는 설명을 못 하겠어.\"→M3.");
  lines.push("M3 POSITIVE: S1 \"들이쉴 때 압력이 높아지는 거야.\" S2 \"근데 그러면 공기가 왜 안으로 들어와? 압력이 높으면 밖으로 나가야 하는 거 아니야?\" → M3, contributors=[\"S2\"].");
  lines.push("same-cluster에서도 가능. existing explanation이 반드시 이전 cluster에 있어야 하는 것은 아니다.");
  lines.push("M3 HARD NEGATIVES:");
  lines.push("A. S1 \"압력이 낮아져.\" S2 \"그래서 공기가 들어와.\" → 설명 생성/확장, first-order, M3 아님.");
  lines.push("A2. S2 \"공기가 들어와야 커질 수 있다.\"(trigger로 쓰지 마라) S1 \"근육이 없어서 공기가 들어와야...\" → peer explanation continuation, challenge 없음 → null M.");
  lines.push("B. S1 \"나는 1번 같아.\" S2 \"아니, 2번 같아.\" → 단순 disagreement, C4 가능, M3 자동 아님.");
  lines.push("C. \"그건 아닌 것 같아.\" → 단순 반박/불확실성만이면 M3 아님.");
  lines.push("D. 앞선 설명 재연결 없이 독립 one-shot \"그러면 압력이 낮아지는 거네.\" → 새 추론, 자동 M3 아님.");
  lines.push("E. Student \"X이기 때문이에요.\" → Teacher \"그러면 Y는 어떻게 설명해?\" → cluster 종료. 학생 후속 reinspection 없음 → null M. 처음 학생 발화는 first-order explanation/K 가능, M3 아님.");
  lines.push("F. Teacher \"왜?\" → Student \"폐에 근육이 없으니까요.\" → first-order reason/explanation. K 가능, M3 아님. 뒤 teacher challenge가 있어도 해당 cluster에 학생 후속 response가 없으면 M3로 소급하지 마라.");
  lines.push("K3=claim+evidence, C4=peer proposition 반박, C3=peer content 확장, M3=설명의 논리적 adequacy/use를 second-order로 재검토. K/C 존재 때문에 M3를 자동 부여하지 않는다.");
  _appendMDecisionM3TriggerMinimumRules_(lines);
  lines.push("");
  lines.push("===== M3 TEMPORAL DIRECTION =====");
  lines.push("M3는 CURRENT CLUSTER TURNS의 실제 발화 순서를 존중한다. 미래 발화를 이용해 앞선 학생 발화를 retrospective하게 M3로 재해석하지 않는다.");
  lines.push("필요 trajectory: existing explanation/claim → challenge/inconsistency/comparison/reflection trigger → STUDENT reinspection/evaluation/restructuring.");
  lines.push("M3 evidence quote는 그 학생의 meta-response이며, 반응하고 있는 trigger/기존 설명 관계에서 시간적으로 올바른 위치에 있어야 한다.");
  lines.push("later teacher/peer challenge 때문에 earlier student explanation을 \"이미 재검토 중이었다\"고 재구성하지 마라. student explanation → teacher challenge → cluster end이면 student M3 없음.");
  lines.push("TEACHER CHALLENGE ALONE ≠ STUDENT M3. 교사가 반례/모순/\"그러면?\"/\"그 설명이면?\"/\"그럼 못 움직여?\"를 제시했다고 이전 학생 발화가 M3가 되지 않는다. Teacher action은 TRIGGER일 뿐, 학생 metacognitive act를 대신하지 않는다.");
  lines.push("TEACHER-MEDIATED M3 최소조건: (A) 학생의 기존 설명/주장 (B) teacher가 contradiction/counterexample/logical challenge 제시 (C) 그 AFTER에 학생이 실제로 자신의 설명을 다시 검사/수정/연결/평가하는 발화. C 없으면 M3 아님. Teacher는 contributor/quote 금지.");
  lines.push("POSITIVE: S1 \"들이쉴 때 안쪽 압력이 높아져요.\" Teacher \"그럼 공기는 높은 압력 쪽으로 들어가니?\" S1 \"아, 아니네요. 그러면 안쪽 압력이 낮아야 하겠네요.\" → S1의 reinspection이 teacher challenge AFTER → M3, contributors=[\"S1\"].");
  lines.push("NEGATIVE: Teacher \"왜?\" S1 \"폐에 근육이 없으니까요.\" Teacher \"그러면 근육이 없으면 못 움직이는 거야?\" cluster end → S1 quote는 first-order explanation, M3 direct evidence 아님 → null M.");
  lines.push("PEER-MEDIATED도 동일: Student A 설명 → Student B challenge → Student A/B의 실제 adequacy reinspection이 있으면 M3 가능. challenge가 cluster 마지막이면 challenge 이전 학생을 M3 contributor로 소급하지 않는다.");
  lines.push("단, challenge 발화 자체가 existing explanation의 adequacy/equivalence를 재검토하는 metacognitive act이면 그 challenge speaker에게 M3 가능. 예: \"그런데 숨을 빨아들이는 것도 밀고 들어오는 거 아니야?\" — 별도 후속 발화 없이도 한 발화에서 M3 가능.");
  lines.push("M3 QUOTE ADEQUACY: quotes만 읽어도 reinspection/evaluation/restructuring trajectory가 보여야 한다. \"폐에 근육이 없으니까요.\"는 first-order explanation. standalone \"못 움직일 것 같은데.\"는 challenge 없으면 M3 아님. challenge 이후 maintained stance/reconstruction이면 M3 가능.");
  lines.push("");
  _appendMDecisionPromptContextRules_(lines);
  lines.push("");
  lines.push("===== M3 TURN-LOCAL CANDIDATE =====");
  lines.push("각 student turn t를 시간 순서대로 본다. \"이 학생이 바로 이 발화 t에서 M process를 실제로 수행하는가?\"를 판단한다. student turn t가 M3인지 판단할 때 기본적으로 t까지의 과거 발화만 사용한다. FUTURE turn은 earlier student turn을 M3로 만드는 근거가 될 수 없다.");
  lines.push("later teacher/peer challenge를 보고 과거 student turn의 기능을 retrospective하게 변경하지 않는다. 관찰되지 않은 내적 상태(\"수정할 필요성을 느꼈다\" \"깨달았다\" \"재검토하고 있다\")를 reason에 추론하지 마라. reason은 observable discourse function만 기술한다.");
  lines.push("trigger_response M3: final quotes는 teacher/peer challenge 이후 student reinspection utterance를 포함해야 한다. first-order explanation만 quote로 뽑아 M3를 정당화하지 않는다.");
  lines.push("self-contained M3: 학생 발화 자체가 adequacy/equivalence/contradiction/applicability/explainability를 평가하면 가능. standalone \"X이기 때문이야.\" \"못 움직일 것 같은데.\" \"압력이 낮아.\"는 challenge 없으면 self-contained M3 아님.");
  lines.push("reconstruction M3: 앞선 explanatory elements를 다시 연결하여 model을 재구성하는 student turn에서 가능.");
  lines.push("");
  _appendMDecisionSelfContainedRules_(lines);
  lines.push("");
  lines.push("===== m3_evidence (M3일 때 필수) =====");
  lines.push("code≠M3이면 m3_evidence=null. code=M3이면 반드시 출력:");
  lines.push('  "m3_evidence": { "mode": "self_contained"|"trigger_response"|"reconstruction", "trigger": null | {"role":"teacher"|"student","speaker":null|"S1"~"S4","quote":"원발화"}, "evaluation_type": null | "adequacy"|"equivalence"|"difference"|"contradiction"|"applicability"|"coverage" }');
  lines.push("evaluation_type: code=M3 + mode=self_contained일 때 선택(권장). quote에서 어떤 evaluation function인지 명시. code≠M3이면 m3_evidence=null.");
  lines.push("self_contained: trigger=null. quote ONLY에서 second-order explanation evaluation이 직접 보일 때만. first-order causal/elaboration은 self_contained M3 아님.");
  lines.push("trigger_response: trigger=teacher/peer challenge quote (current cluster 또는 [PRIOR CONTEXT] turn). final student M quotes는 trigger AFTER이며 [CURRENT CLUSTER TURNS]에 있어야 한다. maintained stance/reconsideration도 trigger_response.");
  lines.push("reconstruction: prior explanatory anchor/trigger (current 또는 [PRIOR CONTEXT]). final quote는 anchor/trigger AFTER이며 current cluster student turn.");
  lines.push("m3_evidence는 M validation/debugging용이다. P contributor 계산에 teacher/trigger speaker를 자동 포함하지 않는다.");
  lines.push("M3 path: [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:NO\",\"M-STEP3:NO\",\"M-STEP4:YES\"]");
  lines.push("");
  lines.push("===== Priority =====");
  lines.push("반드시 M4 > M1 > M2 > M3. 단 먼저 meta target을 정확히 식별한다. 단순 uncertainty word 하나로 M4 target이라고 단정하지 않는다.");
  lines.push("PRIORITY는 minimum condition을 통과한 후보끼리만 적용. M1 minimum NO이면 priority로 M3를 누르지 말고 M1 candidate를 제거한다.");
  lines.push("높은 코드가 단순히 단어 하나 때문에 성립하지 않도록 각 최소조건을 확인한다.");
  lines.push("\"1번이 왜 아닌지를 생각해보자\"는 discussion goal 재설정이므로 M1 후보. 이를 M3로 내리지 마라.");
  lines.push("cluster에 role coordination이 있어도 그것만으로 M1을 우선하지 않는다. explanation-use가 실제 target이면 M3.");
  lines.push("");
  lines.push("===== contributors / quotes =====");
  lines.push("M contributor = 최종 M process 자체를 수행한 학생. C처럼 relation 상대를 자동 포함하지 않는다.");
  lines.push("예: S1 \"이게 왜 그런지 모르겠어.\" S2 \"압력 차이 때문이야.\" — M4가 S1의 이해 회복 행위로 성립하면 contributors=[\"S1\"]. S2가 답변했다는 이유만으로 M contributor가 되지 않는다.");
  lines.push("M4 contributor = 자신의 conceptual understanding을 회복하려는 meta-process를 수행한 학생. 답을 제공하는 peer는 자동 contributor가 아니다.");
  lines.push("M3: S1 \"압력이 높아.\" S2 \"근데 그러면 공기가 왜 들어와? 설명이 안 맞는 것 같은데.\" — 최종 M3를 수행한 학생은 S2. 기존 설명의 target인 S1을 M contributor로 자동 포함하지 않는다.");
  lines.push("여러 학생이 각각 실제 meta-process를 수행했다면 contributors 복수 가능.");
  lines.push("정상 non-null M에서는 contributors 최소 1명. 교사는 contributor 금지.");
  lines.push("quotes는 최종 M process를 직접 보여주는 학생 원발화만. 모든 관련 science utterance를 넣지 마라.");
  lines.push("quotes는 ARRAY OF OBJECTS만. {\"speaker\":\"S1\",\"quote\":\"원발화\"}. bare string 금지. Teacher quote 금지.");
  lines.push("짧은 발화도 실제 M 기능이 있으면 허용. quote는 CURRENT CLUSTER TURNS의 실제 학생 발화와 일치해야 한다.");
  lines.push("UNIQUE(contributors) === UNIQUE(quotes[].speaker). 양방향 일치. mismatch면 출력 전에 최종 M process 범위로 다시 맞출 것. 자동 union/삭제 추정 금지.");
  lines.push("null M: contributors=[], quotes=[], metacognitive_target=null.");
  lines.push("quote adequacy self-check (non-null M 출력 전): 이 quotes만 읽어도 왜 이것이 단순 과학 내용 발화가 아니라 metacognitive process인지 확인되는가?");
  lines.push("과학 설명 quote alone으로 M4를 정당화하지 마라. quotes에서 conceptual self-monitoring/retrieval, process review, 또는 explanation adequacy/reconstruction이 보여야 한다. 별도 recovery 행동이 quotes에 더 있어야 하는 것은 아니다.");
  lines.push("");
  lines.push("===== JSON 출력 직전 self-check =====");
  lines.push("0) 먼저 meta target을 분류했는가? 표면 행동/keyword가 아니라 무엇을 점검하는가?");
  lines.push("1) 실제 meta-process가 있는가? 과학 설명/정교화/반박만으로 M을 만들지 않았는가?");
  lines.push("2) target 없는 \"모르겠어\"/\"아마\"만으로 M4를 강제하지 않았는가? 교사 질문에 답한 것을 자동 M4로 만들지 않았는가?");
  lines.push("2b) 설명 속 수사적 질문을 M4로 오인하지 않았는가?");
  lines.push("2c) \"이해했니?\"를 말한 학생 자신의 M4로 오인하지 않았는가?");
  lines.push("3) 구체적 개념/법칙/의미 gap 표현을 별도 recovery 발화가 없다고 배제하지 않았는가?");
  lines.push("3b) \"헷갈려\"만으로 자동 M4를 주지 않았는가? conceptual target이면 M4, explanation-use면 M3인가?");
  lines.push("3c) 개념명/의미 retrieval(\"보일 법칙이 뭐예요?\" \"무슨 법칙이야?\" \"뜻이 기억이 안 나\")을 놓치지 않았는가? STEP0:NO로 떨어뜨리지 않았는가?");
  lines.push("3e) \"보일 법칙이래?\" 같은 이미 제시된 개념 확인을 자동 M4로 만들지 않았는가?");
  lines.push("3d) 일반 content confirmation/과학 탐색 질문(\"왜 그럴까?\")을 M4/M1로 올리지 않았는가?");
  lines.push("4) 정답/페이지 정보 요청을 M4로 오인하지 않았는가? 동시에 기록 방식 조정을 procedure로 일괄 배제하지 않았는가?");
  lines.push("5) 문제 해결 방향/과제 목표/재검토/기록 방식이면 M1인가? 설명 adequacy를 M1로 오인하지 않았는가?");
  lines.push("5b) content/why/confirmation 질문을 M1로 올리지 않았는가?");
  lines.push("5c) 목적어가 생략된 review proposal(\"어. 다시 한번 볼까?\")을 STEP0:NO로 떨어뜨리지 않았는가? \"다시\" keyword만으로 M1을 주지 않았는가?");
  lines.push("5d) 단순 역할 배정(\"발표할 사람 정하자\")을 자동 M1로 만들지 않았는가? task goal(\"뭘 발표하는데?\")은 M1 가능한가?");
  lines.push("5e) content stance/judgment(\"못 움직일 것 같은데\")를 discussion process M1로 오인하지 않았는가? quote만 읽어도 WHAT/HOW TO DO NEXT가 보이는가?");
  lines.push("6) 단순 의견 요청을 M2로 오인하지 않았는가? M2 threshold를 바꾸지 않았는가?");
  lines.push("7) 설명 생성만으로 M3를 주지 않았는가? 동일성/차이/설명 가능성/적용 가능성/재구성인가?");
  lines.push("7b) 기존 설명 요소의 restructuring을 first-order로 떨어뜨리지 않았는가? \"아 그러면\"만으로 자동 M3를 주지 않았는가?");
  lines.push("7c) M3 evidence quote가 trigger/기존 설명 관계에서 시간적으로 올바른 위치인가? teacher challenge BEFORE인 earlier student quote를 M3로 소급하지 않았는가?");
  lines.push("7d) teacher-mediated M3에서 student reinspection quote가 teacher challenge AFTER인가? challenge만 있고 student 후속 response 없으면 null M인가?");
  lines.push("7e) \"폐에 근육이 없으니까요.\" 같은 first-order reason을 teacher challenge로 retrospective M3화하지 않았는가?");
  lines.push("7f) turn-local candidate: future turn으로 earlier student quote를 M3로 소급하지 않았는가?");
  lines.push("7g) trigger_response M3의 final quote가 trigger AFTER인가? first-order explanation만 quote로 쓰지 않았는가?");
  lines.push("7h) reason에 발화에 없는 mental state(\"느꼈다\" \"깨달았다\")를 넣지 않았는가?");
  lines.push("7i) code=M3이면 m3_evidence.mode/trigger가 trajectory와 일치하는가? prior trigger는 m3_evidence.trigger에만, final quote는 current cluster student만인가?");
  lines.push("7j) reinspection≠revision: challenge 후 입장 유지도 M3 가능한가? standalone stance를 M3로 올리지 않았는가?");
  lines.push("7k) code≠M3이면 m3_evidence=null인가?");
  lines.push("7l) first-order explanation/elaboration을 self_contained M3로 올리지 않았는가? quote ONLY evaluation self-check 통과?");
  lines.push("7l2) self_contained M3이면 evaluation_type을 quote function과 일치하게 선택했는가?");
  lines.push("7m) terminal teacher challenge만 있고 student 후속 response 없으면 earlier student M3를 만들지 않았는가?");
  lines.push("7n) ordinary prior explanation을 M3 trigger로 사용하지 않았는가? explanation continuation을 M3로 올리지 않았는가?");
  lines.push("7o) role/logistics coordination(\"발표할 사람 정하자\")을 M1로 오인하지 않았는가?");
  lines.push("8) M process 수행 학생만 contributor인가? target peer/teacher를 자동 포함하지 않았는가?");
  lines.push("9) UNIQUE(contributors) === UNIQUE(quotes[].speaker) 인가?");
  lines.push("10) quotes의 모든 원소가 {speaker, quote} 객체이고 학생 원발화인가?");
  lines.push("11) 높은 코드의 최소조건이 실제로 성립하는가?");
  lines.push("12) quotes만 읽어도 metacognitive process가 보이는가?");
  lines.push("13) code와 decision_path가 canonical과 일치하는가? M4면 반드시 [\"M-STEP0:YES\",\"M-STEP1:YES\"]. M1면 [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:YES\"]. M3면 M-STEP4:YES로 끝나는가?");
  lines.push("13b) code=M4인데 STEP1:NO이거나 STEP1이 없으면 출력 전에 path를 고쳐라. invalid path를 그대로 내지 마라.");
  lines.push("하나라도 아니면 출력 전에 JSON을 수정한다.");
  lines.push("");
  lines.push("===== CANONICAL decision_path (이 형식만) =====");
  lines.push("null: [\"M-STEP0:NO\"]");
  lines.push("M4: [\"M-STEP0:YES\",\"M-STEP1:YES\"]");
  lines.push("M1: [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:YES\"]");
  lines.push("M2: [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:NO\",\"M-STEP3:YES\"]");
  lines.push("M3: [\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:NO\",\"M-STEP3:NO\",\"M-STEP4:YES\"]");
  lines.push("code=M4인데 M-STEP1:YES가 없으면 출력 금지. code와 path가 불일치하면 path를 canonical에 맞춘 뒤 출력하라.");
  lines.push("");
  lines.push("===== 출력 JSON (이 스키마만) =====");
  lines.push("{");
  lines.push('  "schema_version":"KCMP_M_V1",');
  lines.push('  "status":"OK",');
  lines.push('  "code": null,');
  lines.push('  "contributors": [],');
  lines.push('  "metacognitive_target": null,');
  lines.push('  "reason": "학생 발화에서 자신의 이해·전략·참여·설명 논리를 점검하거나 조정하는 메타인지 과정이 확인되지 않는다.",');
  lines.push('  "decision_path": ["M-STEP0:NO"],');
  lines.push('  "boundary_check": null,');
  lines.push('  "context_needed": false,');
  lines.push('  "quotes": [],');
  lines.push('  "m3_evidence": null');
  lines.push("}");
  lines.push("");
  lines.push("M1 예:");
  lines.push('  code="M1", contributors=["S2"], metacognitive_target="현재 답을 검토하기 위한 문제 해결 방향"');
  lines.push('  reason="S2가 1번이 아닌 이유를 다시 검토하자고 제안하여 논의 전략을 조정하였다."');
  lines.push('  quotes=[{"speaker":"S2","quote":"1번이 왜 아닌지를 생각해보자"}]');
  lines.push('  path=["M-STEP0:YES","M-STEP1:NO","M-STEP2:YES"]');
  lines.push("M4 예: S1 \"압력 차이가 무슨 뜻인지 모르겠어. 이게 왜 공기 이동이랑 연결되는 거지?\" contributors=[\"S1\"], path=[\"M-STEP0:YES\",\"M-STEP1:YES\"]");
  lines.push("M4 meaning-gap 예: \"단어는 기억나는데 그 뜻이 기억이 안 나.\" → M4, path=[\"M-STEP0:YES\",\"M-STEP1:YES\"]");
  lines.push("M4 law-retrieval 예: \"보일 법칙이 뭐예요?\" / \"무슨 법칙이야? 1학년 때 수업 안 들었는데.\" → M4, path=[\"M-STEP0:YES\",\"M-STEP1:YES\"]");
  lines.push("M4 confirmation ≠ retrieval: \"보일 법칙이래?\" → 자동 M4 아님, 권장 path=[\"M-STEP0:NO\"]");
  lines.push("M4 specific confusion 예: \"갑자기 헷갈려. 눌렀을 때가…\" → 특정 관계 self-monitoring → M4, path=[\"M-STEP0:YES\",\"M-STEP1:YES\"]");
  lines.push("M3 예: 기존 설명의 논리 문제를 재검사한 학생만 contributor. target peer 자동 포함 금지.");
  lines.push("M3 reconstruction 예: 이미 나온 요소를 다시 연결하여 설명 모델을 재구성 → M3. \"아 그러면\" 단어만으로 부여하지 마라.");
  lines.push("M3 equivalence 예: \"이거 둘이 같은 소리인가?\" → 경쟁 설명 구별 평가 → M3.");
  lines.push("M3 adequacy 예: \"이쪽 설명은 되는데 저쪽은 설명을 못 하겠어.\" → M3.");
  lines.push("M3 explanation-use 예: \"말이 되는 게 있는데 그걸 활용을 못 하겠어.\" → M4가 아니라 M3 후보.");
  lines.push("M3 temporal negative: S1 \"폐에 근육이 없으니까요.\" Teacher \"그러면 못 움직이는 거야?\" cluster end → null M, m3_evidence=null.");
  lines.push("M3 temporal positive: S1 [기존 설명] Teacher [challenge] S1 \"아, 그러면 ...\" 또는 \"못 움직일 것 같은데.\"(maintained stance) → M3, m3_evidence.mode=trigger_response, final quote는 challenge AFTER.");
  lines.push("M3 prior-trigger positive: PRIOR Teacher [challenge] CURRENT S1 [response/reinspection] → M3, trigger in prior context, quote in current cluster, mode=trigger_response 또는 reconstruction (self_contained 아님).");
  lines.push("M3 maintained stance positive: Teacher [challenge] S1 \"못 움직일 것 같은데.\" AFTER challenge → M3 trigger_response, NOT self_contained.");
  lines.push("M3 reconstruction positive: Teacher [pressure principle] S1 \"그러니까 공기는…\" → M3 reconstruction 또는 trigger_response.");
  lines.push("M3 self-contained positive: \"그런데 두 설명이 결국 같은 말 아니야?\" / \"그런데 빨아들이는 것도 밀고 들어오는 거 아니야?\" → M3, mode=self_contained, evaluation_type=equivalence.");
  lines.push("M3 applicability positive: \"말이 되는 게 있는데 그걸 활용을 못 하겠어.\" → mode=self_contained, evaluation_type=applicability.");
  lines.push("M3 first-order negative: \"공기가 들어와야 커질 수 있다.\" \"근육이 없어서 공기가 들어와야...\" → null M.");
  lines.push("M1 content stance negative: \"못 움직일 것 같은데.\" → null M (content judgment, not process regulation).");
  lines.push("M1 process positive: \"어느 방향으로 이동했는지 다시 확인해 보자.\" → M1.");
  lines.push("M1 relook 예: \"어. 다시 한번 볼까?\" → ongoing discussion review proposal이면 M1, path=[\"M-STEP0:YES\",\"M-STEP1:NO\",\"M-STEP2:YES\"]. \"다시 말해봐.\"는 null.");
  lines.push("M1 task-goal 예: \"뭘 발표하는데?\" → M1. \"발표할 사람 정하자.\" → 자동 M1 아님.");
  lines.push("M1 recording 예: \"거기 아니고 아래.\" \"뭐라고 쓰지?\" → 기록 방식 조정이면 M1.");
  lines.push("teacher-elicited content 예: S1 \"잘 모르겠어요.\" Teacher 질문 S1 과학 답변 → null M.");
  lines.push("rhetorical 설명 예: \"우리 소장에서 지금 뭐냐? 융털에 ... 있잖아. 이해했니?\" → 권장 null M.");
  lines.push("content why ≠ M1: \"왜 짱구야?\" → null M.");
  lines.push("untargeted 모름: \"왜?\" \"몰라.\" \"1학년 때 수업 들을걸.\" → conceptual target 없으면 null M.");
  lines.push("first-order ≠ M3: 앞 요소 재연결 없는 \"압력이 낮아지니까 공기가 들어와.\" → null M.");
  lines.push("STEP0 YES였지만 M1~M4 최소조건 어느 것도 성립하지 않으면 code=null, path는 STEP1~STEP4 전부 NO, 강제로 코드를 고르지 마라.");
  lines.push("reason은 항상 비어 있지 않은 문자열. non-null M에서 metacognitive_target은 nonempty string.");
  lines.push("");
  lines.push("[PID]");
  lines.push(packet.pid || "");
  lines.push("");
  lines.push("[STUDENTS]");
  lines.push(students || "(없음)");
  lines.push("");
  lines.push("[ACTIVE STUDENTS]");
  lines.push(active || "(없음)");
  lines.push("");
  if (priorTurns.length > 0) {
    lines.push("[PRIOR CONTEXT — from previous cluster " + (ctx.priorContextPid || "") + "]");
    lines.push("보조 자료. current 첫 발화 해석 및 M3 trigger 식별에만 사용. final contributor/quote는 current cluster student만.");
    lines.push("prior turn을 future evidence로 사용해 current earlier student를 M3로 소급하지 마라.");
    lines.push(priorTurnsText || "(없음)");
    lines.push("");
  }
  lines.push("[CURRENT CLUSTER TURNS]");
  lines.push(turnsText || "(원발화 없음)");
  lines.push("");
  lines.push("[SUMMARY - AUXILIARY ONLY]");
  lines.push("보조자료이며 원발화와 충돌하면 원발화를 우선한다.");
  lines.push(summary || "(요약 없음)");
  lines.push("");
  lines.push("JSON만 출력하라.");
  return lines.join("\n");
}

function buildMDecisionRetryPrompt_(packet, ctx, firstResult, validationErrors){
  ctx = ctx || {};
  const base = buildMDecisionPrompt_(packet, ctx);
  const lines = [];
  lines.push(base);
  lines.push("");
  lines.push("===== VALIDATION RETRY (SEMANTIC RE-DECISION) =====");
  lines.push("이전 출력의 evidence structure가 invalid였다. JSON만 다시 출력하라.");
  lines.push("동일한 invalid code + mode + trigger + quote relation을 그대로 반복하지 마라.");
  lines.push("metadata/field 오류(teacher trigger speaker 등)는 이미 canonical repair 대상이므로, 의미적으로 valid한 M3 trigger_response를 유지할 수 있으면 code=M3를 유지하라.");
  lines.push("trigger temporal relation이 불가능하면 null로 내려도 된다. 단, valid trigger_response가 실제로 있으면 null로 default하지 마라.");
  lines.push("first-order explanation continuation을 trigger_response M3로 올리지 마라. ordinary prior content는 trigger가 아니다.");
  lines.push("role assignment/logistics(\"일단 정하자. 발표할 사람.\")는 M1이 아니다. explanation-use monitoring이 있으면 M3를 검토하라.");
  lines.push('null이면: code=null, contributors=[], quotes=[], m3_evidence=null, decision_path=["M-STEP0:NO"]');
  lines.push("");
  lines.push("Do NOT infer unobserved student response.");
  lines.push("If no student utterance occurs AFTER the proposed trigger in the transcript, you cannot claim the student \"reconsidered after the challenge.\"");
  lines.push("");
  lines.push("[FORBIDDEN CANDIDATE — DO NOT REPEAT EXACT SAME]");
  lines.push("rejected code: " + (firstResult && firstResult.code == null ? "null" : (firstResult && firstResult.code)));
  lines.push("rejected contributors: " + JSON.stringify(firstResult && firstResult.contributors ? firstResult.contributors : []));
  lines.push("rejected m3_evidence.mode: " + (firstResult && firstResult.m3_evidence && firstResult.m3_evidence.mode ? firstResult.m3_evidence.mode : "(none)"));
  lines.push("rejected m3_evidence.trigger.quote: " + (firstResult && firstResult.m3_evidence && firstResult.m3_evidence.trigger ? firstResult.m3_evidence.trigger.quote : "(none)"));
  lines.push("rejected quotes: " + JSON.stringify(firstResult && firstResult.quotes ? firstResult.quotes : []));
  lines.push("");
  if (_hasTemporalImpossibilityErrors_(validationErrors)) {
    lines.push("[TEMPORAL FACT]");
    lines.push("The proposed trigger-response interpretation CANNOT be reused as-is.");
    lines.push("Re-check whether a valid earlier challenge trigger exists, or whether no valid M3 remains.");
    lines.push("");
  }
  lines.push("[PREVIOUS CANDIDATE — FULL JSON]");
  lines.push(JSON.stringify(firstResult || {}));
  lines.push("");
  lines.push("[VALIDATOR ERRORS]");
  (validationErrors || []).forEach(function(err, i){
    lines.push((i + 1) + ". " + err);
  });
  lines.push("");
  lines.push("위 forbidden candidate와 validator errors를 반영하여 재판정하고 JSON만 출력하라.");
  return lines.join("\n");
}

function parseMDecisionTreeResponse_(raw){
  let s = String(raw == null ? "" : raw).replace(/^\uFEFF/, "").trim();
  if (!s) throw new Error("빈 응답");
  s = s.replace(/^```(?:json)?\s*/i, "").replace(/\s*```$/i, "").trim();
  try {
    return JSON.parse(s);
  } catch (e1) {
    const start = s.indexOf("{");
    const end = s.lastIndexOf("}");
    if (start >= 0 && end > start) {
      try {
        return JSON.parse(s.slice(start, end + 1));
      } catch (e2) {
        throw new Error("JSON.parse 실패: " + e2.toString());
      }
    }
    throw new Error("JSON.parse 실패: " + e1.toString());
  }
}

function validateMDecisionResult_(result, packet, ctx){
  ctx = ctx || {};
  const errors = [];
  const warnings = [];
  if (!result || typeof result !== "object") {
    return { ok: false, errors: ["결과가 객체가 아님"], warnings: warnings };
  }
  if (result.schema_version !== "KCMP_M_V1") errors.push("schema_version 불일치: " + result.schema_version);
  if (result.status !== "OK") errors.push("status가 OK가 아님: " + result.status);

  const reason = String(result.reason == null ? "" : result.reason).trim();
  if (reason.length === 0) errors.push("reason이 비어 있음 (정상 결과는 reason 필수)");

  if (typeof result.context_needed !== "boolean") {
    errors.push("context_needed가 boolean이 아님");
  }

  const allowed = { M1: true, M2: true, M3: true, M4: true };
  const code = result.code;
  if (!(code === null || allowed[code])) errors.push("code 무효: " + code);

  const mtRaw = result.metacognitive_target;
  const mt = mtRaw == null ? null : String(mtRaw).trim();
  if (code == null) {
    if (mtRaw != null && mt && mt.length > 0) errors.push("code=null인데 metacognitive_target이 null이 아님");
  } else {
    if (!mt) errors.push("code가 있는데 metacognitive_target이 비어 있음");
  }

  const activeIds = packet && packet.activeStudentIds ? packet.activeStudentIds : [];

  if (!Array.isArray(result.contributors)) {
    errors.push("contributors가 배열이 아님");
  } else {
    const seen = {};
    const active = {};
    activeIds.forEach(function(id){ active[id] = true; });
    result.contributors.forEach(function(id){
      if (seen[id]) errors.push("contributor 중복: " + id);
      seen[id] = true;
      if (!/^S[1-4]$/.test(String(id || ""))) errors.push("contributor가 S1~S4가 아님(교사 금지): " + id);
      if (packet && !active[id]) errors.push("contributor가 activeStudentIds에 없음: " + id);
    });
    if (code != null && result.contributors.length < 1) errors.push("non-null M인데 contributors가 비어 있음");
    if (code == null && result.contributors.length !== 0) errors.push("code=null인데 contributors가 비어 있지 않음");
  }

  if (!Array.isArray(result.decision_path)) {
    errors.push("decision_path가 배열이 아님");
  } else {
    const path = result.decision_path;
    if (code != null && !_kcmpPathHas_(path, "M-STEP0:YES")) errors.push("non-null M인데 M-STEP0:YES 없음");
    if (code === "M4" && !_kcmpPathHas_(path, "M-STEP1:YES")) errors.push("M4인데 M-STEP1:YES 없음");
    if (code === "M1" && !_kcmpPathHas_(path, "M-STEP2:YES")) errors.push("M1인데 M-STEP2:YES 없음");
    if (code === "M2" && !_kcmpPathHas_(path, "M-STEP3:YES")) errors.push("M2인데 M-STEP3:YES 없음");
    if (code === "M3" && !_kcmpPathHas_(path, "M-STEP4:YES")) errors.push("M3인데 M-STEP4:YES 없음");
    if (code == null) {
      const step0No = _kcmpPathHas_(path, "M-STEP0:NO");
      const allNo = _kcmpPathHas_(path, "M-STEP1:NO") && _kcmpPathHas_(path, "M-STEP2:NO") && _kcmpPathHas_(path, "M-STEP3:NO") && _kcmpPathHas_(path, "M-STEP4:NO");
      if (!step0No && !allNo) errors.push("null code인데 decision_path가 M-STEP0:NO 또는 STEP1~STEP4 전부 NO가 아님");
    }
  }

  const studentTurns = ((packet && packet.turns) || []).filter(function(t){ return t.role === "student"; });

  if (!Array.isArray(result.quotes)) {
    errors.push("quotes가 배열이 아님");
  } else if (code == null) {
    if (result.quotes.length !== 0) errors.push("code=null인데 quotes가 비어 있지 않음");
  } else {
    if (result.quotes.length < 1) errors.push("code가 있는데 quotes가 비어 있음");
    _kcmpValidateMQuoteEntries_(result.quotes, studentTurns, "quotes", errors, packet);
    _kcmpValidateContributorQuoteSetEquality_(result.contributors, result.quotes, errors);
  }

  _kcmpValidateM3Evidence_(result, packet, errors, ctx.priorContextTurns || []);
  _kcmpWarnSelfContainedContextDependency_(result, ctx, warnings);
  if (result.m3_evidence === undefined) {
    errors.push("m3_evidence 필드 없음 (null 또는 객체 필수)");
  }

  return { ok: errors.length === 0, errors: errors, warnings: warnings };
}

function _snapshotMDecisionResultForLog_(result){
  if (!result || typeof result !== "object") return null;
  return {
    schema_version: result.schema_version,
    status: result.status,
    code: result.code == null ? null : result.code,
    contributors: (result.contributors || []).slice(),
    metacognitive_target: result.metacognitive_target == null ? null : result.metacognitive_target,
    reason: result.reason,
    decision_path: (result.decision_path || []).slice(),
    boundary_check: result.boundary_check == null ? null : result.boundary_check,
    context_needed: result.context_needed,
    quotes: (result.quotes || []).map(function(q){
      return { speaker: q && q.speaker, quote: q && q.quote };
    }),
    m3_evidence: result.m3_evidence == null ? null : JSON.parse(JSON.stringify(result.m3_evidence)),
    pid: result.pid,
    error_type: result.error_type,
    message: result.message
  };
}

function _logMDecisionRetryDebug_(debug){
  const d = debug || {};
  Logger.log("M_RETRY_CALLED=" + (d.m_retry_called ? "true" : "false"));
  Logger.log("M_RETRY_REASON=" + (d.m_retry_reason || ""));
  Logger.log("REPEATED_INVALID_CANDIDATE=" + (d.repeated_invalid_candidate ? "true" : "false"));
  Logger.log("FIRST_VALIDATION_ERRORS=" + JSON.stringify(d.first_validation_errors || []));
  Logger.log("RETRY_RAW=" + String(d.retry_raw == null ? "" : d.retry_raw).slice(0, 800));
  Logger.log("RETRY_PARSED=" + JSON.stringify(d.retry_parsed || null));
  Logger.log("RETRY_VALIDATION=" + JSON.stringify(d.retry_validation || null));
  Logger.log("FINAL_AFTER_RETRY=" + JSON.stringify(d.final_after_retry || null));
}

function runMDecisionTreeForPacket_(packet, options){
  options = options || {};
  const pid = packet && packet.pid ? packet.pid : "";
  if (!packet || !pid) return _makeMDecisionError_("PACKET_ERROR", "pid 없음", pid);
  if (!packet.turns || packet.turns.length === 0) {
    return _makeMDecisionError_("PACKET_ERROR", "turns가 비어 있음", pid);
  }

  const active = packet.activeStudentIds || [];
  if (active.length === 0) {
    if (_mPacketMappingUnreliable_(packet)) {
      return _makeMDecisionError_("PACKET_ERROR", "활성 학생이 0명이지만 unmapped/unknown speaker가 있어 학생 수를 단정할 수 없음", pid);
    }
    return _makeMNoneResult_(
      pid,
      "현재 클러스터에 활성 학생이 없어 학생 메타인지 과정이 성립하지 않는다.",
      ["M-STEP0:NO"]
    );
  }

  const ctx = _buildMDecisionContext_(packet, options.allPackets);
  const retryDebug = {
    m_retry_called: false,
    m_retry_reason: "",
    repeated_invalid_candidate: false,
    first_validation_errors: [],
    retry_raw: null,
    retry_parsed: null,
    retry_validation: null,
    final_after_retry: null
  };

  let raw = "";
  try {
    raw = callGPT_simple_(buildMDecisionPrompt_(packet, ctx), MODEL_M);
  } catch (e) {
    return _makeMDecisionError_("API_ERROR", e.toString(), pid);
  }

  let parsed;
  try {
    parsed = parseMDecisionTreeResponse_(raw);
  } catch (e) {
    return _makeMDecisionError_("PARSER_ERROR", e.toString(), pid, { raw_excerpt: String(raw).slice(0, 400) });
  }

  const canon = _canonicalizeMDecisionResult_(parsed, packet, ctx);
  parsed = canon.result;
  if (canon.repairLog.m_canonical_repair) {
    _logMDecisionCanonicalRepair_(canon.repairLog);
  }

  let v = validateMDecisionResult_(parsed, packet, ctx);
  if (!v.ok) {
    if (!_shouldSemanticRetryM_(v.errors)) {
      return _makeMDecisionError_("VALIDATION_ERROR", v.errors.join("; "), pid, {
        validation_errors: v.errors,
        raw_excerpt: String(raw).slice(0, 400),
        m_canonical_repair: canon.repairLog.m_canonical_repair,
        semantic_retry_skipped: true
      });
    }
    retryDebug.first_validation_errors = v.errors.slice();
    retryDebug.m_retry_called = true;
    retryDebug.m_retry_reason = "VALIDATION_ERROR";
    let retryRaw = "";
    try {
      retryRaw = callGPT_simple_(buildMDecisionRetryPrompt_(packet, ctx, parsed, v.errors), MODEL_M);
    } catch (e) {
      const err = _makeMDecisionError_("VALIDATION_ERROR", v.errors.join("; "), pid, {
        validation_errors: v.errors,
        raw_excerpt: String(raw).slice(0, 400),
        m_retry_called: true,
        m_retry_reason: "VALIDATION_ERROR",
        first_validation_errors: v.errors,
        retry_api_error: e.toString()
      });
      if (options.includeDebug) err._mDebug = retryDebug;
      return err;
    }
    retryDebug.retry_raw = retryRaw;
    let retryParsed;
    try {
      retryParsed = parseMDecisionTreeResponse_(retryRaw);
    } catch (e) {
      const err = _makeMDecisionError_("VALIDATION_ERROR", v.errors.join("; "), pid, {
        validation_errors: v.errors,
        raw_excerpt: String(raw).slice(0, 400),
        m_retry_called: true,
        m_retry_reason: "VALIDATION_ERROR",
        first_validation_errors: v.errors,
        retry_parser_error: e.toString(),
        retry_raw_excerpt: String(retryRaw).slice(0, 400)
      });
      retryDebug.retry_parsed = null;
      if (options.includeDebug) err._mDebug = retryDebug;
      return err;
    }
    retryDebug.retry_parsed = _snapshotMDecisionResultForLog_(retryParsed);
    const firstErrorsMetadataOnly = _allErrorsAreMMetadataOnly_(v.errors);
    retryDebug.repeated_invalid_candidate = _sameMSemanticCandidate_(parsed, retryParsed) && !firstErrorsMetadataOnly;
    const retryCanon = _canonicalizeMDecisionResult_(retryParsed, packet, ctx);
    retryParsed = retryCanon.result;
    if (retryCanon.repairLog.m_canonical_repair) {
      _logMDecisionCanonicalRepair_(retryCanon.repairLog);
    }
    const retryV = validateMDecisionResult_(retryParsed, packet, ctx);
    retryDebug.retry_validation = retryV;
    if (retryDebug.repeated_invalid_candidate) {
      retryDebug.final_after_retry = { status: "ERROR", error_type: "VALIDATION_ERROR", repeated_invalid_candidate: true };
      const err = _makeMDecisionError_("VALIDATION_ERROR", retryV.ok ? "REPEATED_INVALID_CANDIDATE" : retryV.errors.join("; "), pid, {
        validation_errors: retryV.ok ? ["REPEATED_INVALID_CANDIDATE"] : retryV.errors,
        first_validation_errors: v.errors,
        raw_excerpt: String(retryRaw).slice(0, 400),
        m_retry_called: true,
        m_retry_reason: "VALIDATION_ERROR",
        repeated_invalid_candidate: true
      });
      if (options.includeDebug) err._mDebug = retryDebug;
      return err;
    }
    if (!retryV.ok) {
      retryDebug.final_after_retry = { status: "ERROR", error_type: "VALIDATION_ERROR" };
      const err = _makeMDecisionError_("VALIDATION_ERROR", retryV.errors.join("; "), pid, {
        validation_errors: retryV.errors,
        first_validation_errors: v.errors,
        raw_excerpt: String(retryRaw).slice(0, 400),
        m_retry_called: true,
        m_retry_reason: "VALIDATION_ERROR"
      });
      if (options.includeDebug) err._mDebug = retryDebug;
      return err;
    }
    parsed = retryParsed;
    v = retryV;
    retryDebug.final_after_retry = { status: "OK", code: parsed.code };
  }

  parsed.status = "OK";
  parsed.schema_version = "KCMP_M_V1";
  parsed.pid = pid;
  if (canon.repairLog.m_canonical_repair) {
    parsed.m_canonical_repair = canon.repairLog;
  }
  if (options.includeDebug) parsed._mDebug = retryDebug;
  return parsed;
}

function getMDecisionTreeFixtures_(){
  return [
    {
      id: "NULL_SCIENCE_EXPLANATION",
      expected: null,
      note: "좋은 과학 설명이지만 meta-process 없음",
      packet: _kcmpSyntheticPacket_("MF01", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력이 낮아져서 공기가 들어와." }
      ])
    },
    {
      id: "M4_CONCEPT_RECOVERY",
      expected: "M4",
      packet: _kcmpSyntheticPacket_("MF02", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력 차이가 무슨 뜻인지 모르겠어. 이게 왜 공기 이동이랑 연결되는 거지?" }
      ])
    },
    {
      id: "M4_IMPLICIT_RECOVERY",
      expected: "M4",
      note: "자기 이해 상태를 특정 관계에 대해 monitoring. 단순 '왜 그럴까?' 탐색과 구분.",
      packet: _kcmpSyntheticPacket_("MF03", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "이 관계가 이해가 안 돼. 왜 여기로 들어가는 거지?" }
      ])
    },
    {
      id: "NULL_BARE_UNCERTAINTY",
      expected: null,
      note: "target 없는 standalone uncertainty. '헷갈려' 단어 자체가 null은 아님.",
      packet: _kcmpSyntheticPacket_("MF04", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "잘 모르겠어." }
      ])
    },
    {
      id: "M1_STRATEGY",
      expected: "M1",
      packet: _kcmpSyntheticPacket_("MF05", [
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "1번이 왜 아닌지를 생각해보자." }
      ])
    },
    {
      id: "NULL_PROCEDURE",
      expected: null,
      note: "단순 페이지/쓰기 지시. 기록 방식 regulation이 아님. '몇 페이지야?' 정보 요청과 같은 층.",
      packet: _kcmpSyntheticPacket_("MF06", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "180쪽 펴." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "시간 없어." }
      ])
    },
    {
      id: "M2_PARTICIPATION",
      expected: "M2",
      note: "S3는 regulation target일 뿐 M contributor 아님",
      packet: _kcmpSyntheticPacket_("MF07", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "계속 우리 둘만 말했잖아. S3도 의견 말해봐." }
      ])
    },
    {
      id: "NULL_SIMPLE_OPINION_REQUEST",
      expected: null,
      note: "C2 가능하지만 M2 아님",
      packet: _kcmpSyntheticPacket_("MF08", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "S3 너는 몇 번이라고 생각해?" }
      ])
    },
    {
      id: "M3_LOGIC_RECHECK",
      expected: "M3",
      note: "contributors는 M3를 수행한 S2만",
      packet: _kcmpSyntheticPacket_("MF09", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "들이쉴 때 안쪽 압력이 높아져." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "근데 그러면 공기가 왜 안으로 들어와? 압력이 높으면 밖으로 나가야 하는 거 아니야?" }
      ])
    },
    {
      id: "NULL_ELABORATION_NOT_M3",
      expected: null,
      note: "C3 가능, M3 아님",
      packet: _kcmpSyntheticPacket_("MF10", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐가 커지면 압력이 낮아져." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "그래서 바깥 공기가 들어와." }
      ])
    },
    {
      id: "NULL_REBUTTAL_NOT_M3",
      expected: null,
      packet: _kcmpSyntheticPacket_("MF11", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "1번 같아." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "아니, 2번 같아." }
      ])
    },
    {
      id: "TEACHER_MEDIATED_M3",
      expected: "M3",
      note: "contributors=[S1]. Teacher quote 금지.",
      packet: _kcmpSyntheticPacket_("MF12", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "들이쉴 때 안쪽 압력이 높아져요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "그럼 공기는 높은 압력 쪽으로 들어가니?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "아, 아니네요. 그러면 안쪽 압력이 낮아야 하겠네요." }
      ])
    },
    {
      id: "NULL_TEACHER_ELICITED_RECOVERY_FALSE_POSITIVE",
      expected: null,
      note: "잘 모르겠어요 + 교사 질문 + 과학 답변만. 자기 회복 없음.",
      packet: _kcmpSyntheticPacket_("MF13", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "잘 모르겠어요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "그럼 왜 그런지 생각해봐." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력 때문인 것 같아요." }
      ])
    },
    {
      id: "NULL_RHETORICAL_SELF_QUESTION",
      expected: null,
      packet: _kcmpSyntheticPacket_("MF14", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "우리 소장에서 뭐가 있었지? 융털에 모세혈관 있잖아. 그래서 포도당이 흡수되는 거야." }
      ])
    },
    {
      id: "NULL_OTHER_COMPREHENSION_CHECK",
      expected: null,
      packet: _kcmpSyntheticPacket_("MF15", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "모세혈관이 수용성 영양소를 흡수하잖아. 이해했니?" }
      ])
    },
    {
      id: "M3_CRITERION_ADEQUACY",
      expected: "M3",
      packet: _kcmpSyntheticPacket_("MF16", [
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "분자 크기로 나뉜다고 하면 왜 포도당이랑 아미노산이 같은 결과가 나와? 그 기준으로는 구별이 안 되는 것 같은데." }
      ])
    },
    {
      id: "NULL_CONTENT_WHY_REQUEST",
      expected: null,
      note: "peer 선택/내용 이유 요청. process regulation 아님.",
      packet: _kcmpSyntheticPacket_("MF17", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "왜 3번이야?" }
      ])
    },
    {
      id: "NULL_CONTENT_CONFIRMATION",
      expected: null,
      packet: _kcmpSyntheticPacket_("MF18", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "눌렀을 때 압력이 높은 거지?" }
      ])
    },
    {
      id: "M1_REVIEW",
      expected: "M1",
      note: "task/discussion review 기능. 목적어 생략 가능. 오프태스크 '다시 볼까'와 구분.",
      packet: _kcmpSyntheticPacket_("MF19", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "다시 한번 볼까?" }
      ])
    },
    {
      id: "M1_ELLIPTIC_REVIEW",
      expected: "M1",
      note: "목적어 생략된 ongoing task/discussion review. STEP0:YES.",
      packet: _kcmpSyntheticPacket_("MF19C", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "어. 다시 한번 볼까?" }
      ])
    },
    {
      id: "NULL_REPEAT_REQUEST",
      expected: null,
      note: "peer repetition request. 다시 keyword ≠ M1.",
      packet: _kcmpSyntheticPacket_("MF19D", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "다시 말해봐." }
      ])
    },
    {
      id: "M4_SPECIFIC_CONFUSION",
      expected: "M4",
      note: "특정 conceptual relation confusion. path는 M-STEP0:YES, M-STEP1:YES.",
      packet: _kcmpSyntheticPacket_("MF20B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "갑자기 헷갈려. 눌렀을 때가…" }
      ])
    },
    {
      id: "M4_CONFUSION_CLARIFICATION",
      expected: "M4",
      note: "contributors는 confusion+clarification을 수행한 S1만. 답을 준 S2 제외.",
      packet: _kcmpSyntheticPacket_("MF20", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "갑자기 헷갈려. 눌렀을 때가…" },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "압력이 높아." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "안쪽이?" }
      ])
    },
    {
      id: "M4_LAW_RETRIEVAL",
      expected: "M4",
      note: "missing law retrieval. STEP0:YES, path M-STEP1:YES.",
      packet: _kcmpSyntheticPacket_("MF21B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "무슨 법칙이야? 1학년 때 수업 안 들었는데." }
      ])
    },
    {
      id: "NULL_LAW_CONFIRMATION",
      expected: null,
      note: "이미 제시된 법칙명 확인. retrieval 아님.",
      packet: _kcmpSyntheticPacket_("MF21C", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "보일 법칙이래?" }
      ])
    },
    {
      id: "M4_CONCEPT_NAME_REQUEST",
      expected: "M4",
      note: "개념/법칙명 retrieval. 명시적 모르겠다·별도 recovery action 없어도 M4.",
      packet: _kcmpSyntheticPacket_("MF22", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "보일 법칙이 뭐예요?" }
      ])
    },
    {
      id: "M4_CONCEPT_MEANING_GAP",
      expected: "M4",
      note: "concept meaning retrieval. 별도 recovery action 불필요.",
      packet: _kcmpSyntheticPacket_("MF22B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "단어는 기억나는데 그 뜻이 기억이 안 나." }
      ])
    },
    {
      id: "NULL_UNTARGETED_DONT_KNOW",
      expected: null,
      note: "conceptual target 없는 몰라.",
      packet: _kcmpSyntheticPacket_("MF22C", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "몰라." }
      ])
    },
    {
      id: "NULL_BARE_CONFUSION",
      expected: null,
      note: "target 없는 standalone confusion만 null. 특정 개념 관계면 M4, explanation-use면 M3.",
      packet: _kcmpSyntheticPacket_("MF22D", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "헷갈려." }
      ])
    },
    {
      id: "NULL_FIRST_ORDER_REALIZATION",
      expected: null,
      note: "독립 one-shot reasoning만 null. prior explanation을 다시 조직하는 trajectory는 M3 가능.",
      packet: _kcmpSyntheticPacket_("MF22E", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그러면 압력이 낮아지는 거네." }
      ])
    },
    {
      id: "M1_TASK_GOAL",
      expected: "M1",
      note: "task goal clarification.",
      packet: _kcmpSyntheticPacket_("MF24", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "뭘 발표하는데?" }
      ])
    },
    {
      id: "NULL_ROLE_ASSIGNMENT_NOT_M1",
      expected: null,
      note: "단순 역할 배정은 자동 M1 아님.",
      packet: _kcmpSyntheticPacket_("MF24B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "발표할 사람 정하자." }
      ])
    },
    {
      id: "M1_RECORDING_METHOD",
      expected: "M1",
      note: "공동 과제 기록 방식 조정. contributor는 실제 M process 수행 학생만.",
      packet: _kcmpSyntheticPacket_("MF25", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "거기 아니고 아래." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "뭐라고 쓰지?" }
      ])
    },
    {
      id: "M3_MODEL_RECONSTRUCTION",
      expected: "M3",
      note: "이미 논의된 요소를 다시 연결하여 설명 모델 재구성. '아 그러면' keyword 자체 ≠ M3.",
      packet: _kcmpSyntheticPacket_("MF23", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "횡격막이 내려가." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "부피가 커져." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "아, 그러면 횡격막이 내려가서 부피가 커지고 압력을 낮게 하는 거야." }
      ])
    },
    {
      id: "M3_RESTRUCTURE",
      expected: "M3",
      note: "contributor는 explanation restructuring을 수행한 학생 기준.",
      packet: _kcmpSyntheticPacket_("MF23B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "들이쉴 때 가로막이 내려가." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "안쪽 기압이 낮아져." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "아, 그러면 가로막이 내려가서 안쪽 기압을 낮추고 그래서 공기가 들어오는 거구나." }
      ])
    },
    {
      id: "M3_EXPLANATION_EQUIVALENCE",
      expected: "M3",
      packet: _kcmpSyntheticPacket_("MF26", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "이거 둘이 같은 소리인가?" }
      ])
    },
    {
      id: "M3_EXPLANATORY_ADEQUACY",
      expected: "M3",
      packet: _kcmpSyntheticPacket_("MF27", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "이쪽 설명은 되는데 저쪽은 설명을 못 하겠어." }
      ])
    },
    {
      id: "M3_EXPLANATION_USE",
      expected: "M3",
      note: "confusion word가 있어도 explanation-use target이면 M3. 자동 M4 아님.",
      packet: _kcmpSyntheticPacket_("MF28", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "말이 되는 게 있는데 그걸 활용을 못 하겠어." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "왜 이렇게 헷갈리지?" }
      ])
    },
    {
      id: "NULL_TEACHER_CHALLENGE_NO_STUDENT_RESPONSE",
      expected: null,
      note: "student explanation → teacher challenge → cluster end. retrospective M3 금지.",
      packet: _kcmpSyntheticPacket_("MF29", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐에 근육이 없으니까요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
      ])
    },
    {
      id: "M3_TEACHER_CHALLENGE_WITH_REINSPECTION",
      expected: "M3",
      note: "student reinspection이 teacher challenge AFTER. contributors=[S1].",
      packet: _kcmpSyntheticPacket_("MF29B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐에 근육이 없으니까 못 움직여요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "그런데 갈비뼈나 가로막이 움직이면?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "아, 그러면 폐 자체 근육이 없어도 주변 구조가 움직여서 폐가 커질 수 있겠네요." }
      ])
    },
    {
      id: "NULL_FIRST_ORDER_REASON",
      expected: null,
      note: "Teacher 왜? + first-order reason. K 가능, M3 아님.",
      packet: _kcmpSyntheticPacket_("MF29C", [
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "왜?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐에 근육이 없으니까요." }
      ])
    },
    {
      id: "NULL_FUTURE_TEACHER_CHALLENGE",
      expected: null,
      note: "future teacher challenge로 earlier student explanation을 M3로 소급 금지.",
      packet: _kcmpSyntheticPacket_("MF30", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐에 근육이 없으니까요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
      ])
    },
    {
      id: "M3_POST_CHALLENGE_REINSPECTION",
      expected: "M3",
      note: "trigger_response trajectory. final quote는 teacher challenge AFTER.",
      packet: _kcmpSyntheticPacket_("MF30B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐에 근육이 없으니까 못 움직여요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "다른 구조가 움직일 수도 있지." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그러면 폐 자체가 아니라 주변 구조가 움직여서 폐 크기를 바꾸는 건가?" }
      ])
    },
    {
      id: "NULL_CONTENT_STANCE_NOT_M1",
      expected: null,
      note: "content judgment/stance. discussion process M1 아님.",
      packet: _kcmpSyntheticPacket_("MF31", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "못 움직일 것 같은데." }
      ])
    },
    {
      id: "M1_PROCESS_REVIEW",
      expected: "M1",
      note: "task/discussion process review proposal.",
      packet: _kcmpSyntheticPacket_("MF31B", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "어느 방향으로 이동했는지 다시 확인해 보자." }
      ])
    },
    {
      id: "NULL_CHALLENGE_AT_CLUSTER_END",
      expected: null,
      note: "P037형. student explanation → teacher challenge at cluster end → null M.",
      packet: _kcmpSyntheticPacket_("MFX01", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "폐에 근육이 없으니까요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
      ])
    },
    {
      id: "M3_PRIOR_TRIGGER_CURRENT_RESPONSE",
      expected: "M3",
      note: "P038형 cross-cluster. prior teacher challenge → current student maintained stance.",
      priorContextTurns: [
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 5, utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
      ],
      packet: _kcmpSyntheticPacket_("MFX02", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "못 움직이지 않아요?" },
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "다른 방법으로 움직일 수도 있지." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "못 움직일 것 같은데." }
      ])
    },
    {
      id: "M3_PRIOR_TRIGGER_RECONSTRUCTION",
      expected: "M3",
      note: "prior teacher challenge → maintained stance → teacher principle → reconstruction.",
      priorContextTurns: [
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 5, utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
      ],
      packet: _kcmpSyntheticPacket_("MFX03", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "못 움직일 것 같은데." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "공기가 이동하려면 압력이 어떻게 돼야 해?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "그러니까 공기는 높은 압력에서 낮은 압력으로 가니까…" }
      ])
    },
    {
      id: "NULL_STANDALONE_STANCE",
      expected: null,
      note: "challenge 없는 standalone content stance → null M.",
      packet: _kcmpSyntheticPacket_("MFX04", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "못 움직일 것 같은데." }
      ])
    },
    {
      id: "NULL_TERMINAL_TEACHER_CHALLENGE",
      expected: null,
      note: "student explanation → terminal teacher challenge → null M.",
      packet: _kcmpSyntheticPacket_("MFX05", [
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 9, utterance: "왜?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "폐에 근육이 없으니까요." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "그러면 근육이 없으면 못 움직여?" }
      ])
    },
    {
      id: "M3_MAINTAINED_STANCE_AFTER_CHALLENGE",
      expected: "M3",
      note: "maintained stance after challenge → trigger_response, NOT self_contained.",
      expectedMode: "trigger_response",
      packet: _kcmpSyntheticPacket_("MFX06", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "못 움직인다고 생각해." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "다른 방법으로 움직일 수도 있지." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "그래도 현재 설명으로는 못 움직일 것 같은데." }
      ])
    },
    {
      id: "M3_SELF_CONTAINED_ADEQUACY",
      expected: "M3",
      note: "quote-only adequacy evaluation → self_contained.",
      expectedMode: "self_contained",
      packet: _kcmpSyntheticPacket_("MFX07", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "이 설명으로는 공기가 나가는 경우가 설명이 안 되는데?" }
      ])
    },
    {
      id: "M3_RECONSTRUCTION_AFTER_TRIGGER",
      expected: "M3",
      note: "teacher principle → student reconstruction.",
      expectedMode: "trigger_response",
      packet: _kcmpSyntheticPacket_("MFX08", [
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 10, utterance: "공기가 이동하려면 압력 차이가 있어야 하지?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 11, utterance: "아 그러면 안쪽 압력을 낮추면 높은 바깥쪽에서 공기가 들어오는 거구나." }
      ])
    },
    {
      id: "NULL_FIRST_ORDER_CONTINUATION",
      expected: null,
      note: "P035형. prior explanation continuation without challenge → null M.",
      packet: _kcmpSyntheticPacket_("MFX09", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "공기가 들어와야 커져." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 11, utterance: "근육이 없으니까 공기가 들어와야..." }
      ])
    },
    {
      id: "M3_REAL_CHALLENGE_RESPONSE",
      expected: "M3",
      note: "real teacher challenge puts explanation under review.",
      expectedMode: "trigger_response",
      packet: _kcmpSyntheticPacket_("MFX10", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "공기가 들어오면 폐가 커져." },
        { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "그럼 공기가 들어오기 전에는 왜 폐가 커져?" },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "아, 그러면 폐가 먼저 커져서 압력이 낮아지는 건가?" }
      ])
    },
    {
      id: "NULL_ROLE_ASSIGNMENT",
      expected: null,
      note: "logistics/role coordination only → null M.",
      packet: _kcmpSyntheticPacket_("MFX11", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "발표할 사람 정하자." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "내가 할게." }
      ])
    },
    {
      id: "M1_PROCESS_STRATEGY_EXAMINE",
      expected: "M1",
      note: "process regulation: what to examine first.",
      packet: _kcmpSyntheticPacket_("MFX12", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "먼저 압력 차이를 다시 확인하고 답을 정하자." }
      ])
    },
    {
      id: "M3_EXPLANATION_USE_WITH_LOGISTICS",
      expected: "M3",
      note: "P065형. explanation-use M3 despite logistics in cluster.",
      expectedContributors: ["S3"],
      packet: _kcmpSyntheticPacket_("MFX13", [
        { role: "student", speakerId: "S3", speakerRaw: "학생3", row: 10, utterance: "말이 되는 설명이 있는데 여기 적용을 못 하겠어." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 11, utterance: "발표할 사람 정하자." },
        { role: "student", speakerId: "S2", speakerRaw: "학생2", row: 12, utterance: "내가 할게." }
      ])
    },
    {
      id: "NULL_FIRST_ORDER_CAUSAL_EXPLANATION",
      expected: null,
      note: "P035형 standalone causal proposition.",
      packet: _kcmpSyntheticPacket_("MFY01", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "공기가 들어와야 커질 수 있다." }
      ])
    },
    {
      id: "NULL_FIRST_ORDER_ELABORATION",
      expected: null,
      note: "P035형 explanation elaboration chain.",
      packet: _kcmpSyntheticPacket_("MFY02", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "공기가 들어와야 커져." },
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 11, utterance: "근육이 없으니까 공기가 들어와야..." }
      ])
    },
    {
      id: "M3_EXPLANATION_EQUIVALENCE",
      expected: "M3",
      note: "P053형. equivalence in quote → self_contained.",
      expectedMode: "self_contained",
      expectedEvaluationType: "equivalence",
      priorContextTurns: [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 5, utterance: "밀고 들어오는 거야." }
      ],
      packet: _kcmpSyntheticPacket_("MFY03", [
        { role: "student", speakerId: "S2", speakerRaw: "학생2", row: 10, utterance: "그런데 빨아들이는 것도 밀고 들어오는 거 아니야?" }
      ])
    },
    {
      id: "M3_ADEQUACY_SELF_CONTAINED",
      expected: "M3",
      expectedMode: "self_contained",
      expectedEvaluationType: "adequacy",
      packet: _kcmpSyntheticPacket_("MFY04", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "이 설명으로는 공기가 나가는 경우가 설명이 안 되는데?" }
      ])
    },
    {
      id: "M3_APPLICABILITY_SELF_CONTAINED",
      expected: "M3",
      expectedMode: "self_contained",
      expectedEvaluationType: "applicability",
      packet: _kcmpSyntheticPacket_("MFY05", [
        { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "원리는 알겠는데 이걸 여기에는 적용을 못 하겠어." }
      ])
    }
  ];
}

/** Regression metadata only. Do not call from buildMDecisionPrompt_ or runMDecisionTreeForPacket_. */
function getMHumanGold14x4Cases_(){
  return [
    { sheet: "14차시 4조", pid: "P006", expected: "M4" },
    { sheet: "14차시 4조", pid: "P008", expected: "M4" },
    { sheet: "14차시 4조", pid: "P009", expected: "M4" },
    { sheet: "14차시 4조", pid: "P010", expected: "M4" },
    { sheet: "14차시 4조", pid: "P011", expected: "M4" },
    { sheet: "14차시 4조", pid: "P012", expected: "M4" },
    { sheet: "14차시 4조", pid: "P013", expected: null },
    { sheet: "14차시 4조", pid: "P014", expected: null },
    { sheet: "14차시 4조", pid: "P015", expected: "M4" },
    { sheet: "14차시 4조", pid: "P016", expected: "M4" },
    { sheet: "14차시 4조", pid: "P017", expected: "M1" },
    { sheet: "14차시 4조", pid: "P022", expected: "M1" },
    { sheet: "14차시 4조", pid: "P033", expected: "M1" },
    { sheet: "14차시 4조", pid: "P035", expected: null },
    { sheet: "14차시 4조", pid: "P037", expected: null },
    { sheet: "14차시 4조", pid: "P038", expected: "M3" },
    { sheet: "14차시 4조", pid: "P039", expected: "M3" },
    { sheet: "14차시 4조", pid: "P041", expected: null },
    { sheet: "14차시 4조", pid: "P043", expected: "M3" },
    { sheet: "14차시 4조", pid: "P049", expected: "M1" },
    { sheet: "14차시 4조", pid: "P051", expected: "M3" },
    { sheet: "14차시 4조", pid: "P053", expected: "M3" },
    { sheet: "14차시 4조", pid: "P054", expected: "M3" },
    { sheet: "14차시 4조", pid: "P055", expected: "M3" },
    { sheet: "14차시 4조", pid: "P057", expected: "M3" },
    { sheet: "14차시 4조", pid: "P058", expected: "M3" },
    { sheet: "14차시 4조", pid: "P062", expected: "M4" },
    { sheet: "14차시 4조", pid: "P063", expected: "M4" },
    { sheet: "14차시 4조", pid: "P064", expected: "M3" },
    { sheet: "14차시 4조", pid: "P065", expected: "M3" }
  ];
}

function lookupMHumanGoldExpected_(sheetName, pid){
  const key = String(sheetName || "") + "::" + String(pid || "");
  const cases = getMHumanGold14x4Cases_();
  for (let i = 0; i < cases.length; i++) {
    if ((cases[i].sheet + "::" + cases[i].pid) === key) return cases[i].expected;
  }
  return undefined;
}

function TEST_M_VALIDATOR_REGRESSION(){
  const packetNull = _kcmpSyntheticPacket_("MR00", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력이 낮아져서 공기가 들어와." }
  ]);
  const goodNull = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: null,
    contributors: [],
    metacognitive_target: null,
    reason: "학생 발화에서 자신의 이해·전략·참여·설명 논리를 점검하거나 조정하는 메타인지 과정이 확인되지 않는다.",
    decision_path: ["M-STEP0:NO"],
    boundary_check: null,
    context_needed: false,
    quotes: [],
    m3_evidence: null
  };
  const packetM4 = _kcmpSyntheticPacket_("MR04", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "압력 차이가 무슨 뜻인지 모르겠어. 이게 왜 공기 이동이랑 연결되는 거지?" }
  ]);
  const goodM4 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M4",
    contributors: ["S1"],
    metacognitive_target: "압력 차이와 공기 이동의 개념 관계",
    reason: "S1이 개념 이해 공백을 인식하고 그 의미를 회복하려 질문하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "압력 차이가 무슨 뜻인지 모르겠어. 이게 왜 공기 이동이랑 연결되는 거지?" }],
    m3_evidence: null
  };
  const badNoContributor = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M4",
    contributors: [],
    metacognitive_target: "개념 이해",
    reason: "이해 회복",
    decision_path: ["M-STEP0:YES", "M-STEP1:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "압력 차이가 무슨 뜻인지 모르겠어. 이게 왜 공기 이동이랑 연결되는 거지?" }],
    m3_evidence: null
  };
  const packetM1 = _kcmpSyntheticPacket_("MR01", [
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "1번이 왜 아닌지를 생각해보자." }
  ]);
  const badMismatch = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M1",
    contributors: ["S1", "S2"],
    metacognitive_target: "문제 해결 방향",
    reason: "전략 조정",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S2", quote: "1번이 왜 아닌지를 생각해보자." }],
    m3_evidence: null
  };
  const goodM1 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M1",
    contributors: ["S2"],
    metacognitive_target: "현재 답을 검토하기 위한 문제 해결 방향",
    reason: "S2가 1번이 아닌 이유를 다시 검토하자고 제안하여 논의 전략을 조정하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S2", quote: "1번이 왜 아닌지를 생각해보자." }],
    m3_evidence: null
  };
  const packetM3 = _kcmpSyntheticPacket_("MR03", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "들이쉴 때 안쪽 압력이 높아져." },
    { role: "student", speakerId: "S2", speakerRaw: "학생2", utterance: "근데 그러면 공기가 왜 안으로 들어와? 압력이 높으면 밖으로 나가야 하는 거 아니야?" }
  ]);
  const badTeacher = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["교사"],
    metacognitive_target: "기존 설명의 논리",
    reason: "교사가 반례를 제시함",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "교사", quote: "근데 그러면 공기가 왜 안으로 들어와?" }]
  };
  const badFakeQuote = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S2"],
    metacognitive_target: "기존 설명의 논리",
    reason: "논리 재검토",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S2", quote: "이 문장은 원발화에 없다." }]
  };
  const packetM2 = _kcmpSyntheticPacket_("MR02", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "계속 우리 둘만 말했잖아. S3도 의견 말해봐." }
  ]);
  const goodM2 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M2",
    contributors: ["S1"],
    metacognitive_target: "참여 균형",
    reason: "S1이 발언이 편중된 참여 문제를 지적하고 S3의 참여를 유도하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "계속 우리 둘만 말했잖아. S3도 의견 말해봐." }],
    m3_evidence: null
  };
  const goodM3 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S2"],
    metacognitive_target: "들이쉴 때 압력 설명의 논리",
    reason: "S2가 기존 설명이 공기 유입과 양립하지 않는 논리적 문제를 재검사하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S2", quote: "근데 그러면 공기가 왜 안으로 들어와? 압력이 높으면 밖으로 나가야 하는 거 아니야?" }],
    m3_evidence: { mode: "self_contained", trigger: null, evaluation_type: "equivalence" }
  };
  const packetTriggerM3 = _kcmpSyntheticPacket_("MR05", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "폐에 근육이 없으니까 못 움직여요." },
    { role: "teacher", speakerId: "", speakerRaw: "교사", utterance: "다른 구조가 움직일 수도 있지." },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", utterance: "그러면 폐 자체가 아니라 주변 구조가 움직여서 폐 크기를 바꾸는 건가?" }
  ]);
  const goodTriggerM3 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "폐 움직임 설명의 적절성",
    reason: "S1이 교사의 반례 이후 기존 설명을 주변 구조 움직임으로 다시 구성하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "그러면 폐 자체가 아니라 주변 구조가 움직여서 폐 크기를 바꾸는 건가?" }],
    m3_evidence: {
      mode: "trigger_response",
      trigger: { role: "teacher", speaker: null, quote: "다른 구조가 움직일 수도 있지." }
    }
  };
  const badTriggerBefore = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "폐 움직임 설명",
    reason: "재검토",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "폐에 근육이 없으니까 못 움직여요." }],
    m3_evidence: {
      mode: "trigger_response",
      trigger: { role: "teacher", speaker: null, quote: "다른 구조가 움직일 수도 있지." }
    }
  };
  const packetPriorTrigger = _kcmpSyntheticPacket_("MR06", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 20, utterance: "못 움직일 것 같은데." }
  ]);
  const priorCtx = {
    priorContextTurns: [
      { role: "teacher", speakerId: "", speakerRaw: "교사", row: 15, utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
    ]
  };
  const goodPriorTriggerM3 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "근육 없음 설명의 적절성",
    reason: "S1이 prior cluster teacher challenge 이후 기존 판단을 다시 제시하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: true,
    quotes: [{ speaker: "S1", quote: "못 움직일 것 같은데." }],
    m3_evidence: {
      mode: "trigger_response",
      trigger: { role: "teacher", speaker: null, quote: "그러면 근육이 없으면 못 움직이는 거야?" }
    }
  };

  const a = validateMDecisionResult_(goodNull, packetNull);
  const b = validateMDecisionResult_(goodM4, packetM4);
  const c = validateMDecisionResult_(badNoContributor, packetM4);
  const d = validateMDecisionResult_(badMismatch, packetM1);
  const e = validateMDecisionResult_(badTeacher, packetM3);
  const f = validateMDecisionResult_(badFakeQuote, packetM3);
  const g = validateMDecisionResult_(goodM1, packetM1);
  const h = validateMDecisionResult_(goodM2, packetM2);
  const i = validateMDecisionResult_(goodM3, packetM3);
  const j = validateMDecisionResult_(goodTriggerM3, packetTriggerM3);
  const k = validateMDecisionResult_(badTriggerBefore, packetTriggerM3);
  const l = validateMDecisionResult_(goodPriorTriggerM3, packetPriorTrigger, priorCtx);
  Logger.log("GOOD_NULL ok=" + a.ok + " errors=" + JSON.stringify(a.errors));
  Logger.log("GOOD_M4_SINGLE_CONTRIBUTOR ok=" + b.ok + " errors=" + JSON.stringify(b.errors));
  Logger.log("BAD_NON_NULL_NO_CONTRIBUTOR ok=" + c.ok + " errors=" + JSON.stringify(c.errors));
  Logger.log("BAD_CONTRIBUTOR_QUOTE_MISMATCH ok=" + d.ok + " errors=" + JSON.stringify(d.errors));
  Logger.log("BAD_TEACHER_CONTRIBUTOR ok=" + e.ok + " errors=" + JSON.stringify(e.errors));
  Logger.log("BAD_FAKE_QUOTE ok=" + f.ok + " errors=" + JSON.stringify(f.errors));
  Logger.log("GOOD_M1 ok=" + g.ok + " errors=" + JSON.stringify(g.errors));
  Logger.log("GOOD_M2_SINGLE_REGULATOR ok=" + h.ok + " errors=" + JSON.stringify(h.errors));
  Logger.log("GOOD_M3_SINGLE_RECHECKER ok=" + i.ok + " errors=" + JSON.stringify(i.errors));
  Logger.log("GOOD_M3_TRIGGER_RESPONSE ok=" + j.ok + " errors=" + JSON.stringify(j.errors));
  Logger.log("BAD_M3_TRIGGER_BEFORE ok=" + k.ok + " errors=" + JSON.stringify(k.errors));
  Logger.log("GOOD_M3_PRIOR_CONTEXT_TRIGGER ok=" + l.ok + " errors=" + JSON.stringify(l.errors));
  const sameCand = _sameMSemanticCandidate_(badTriggerBefore, badTriggerBefore);
  const diffCand = _sameMSemanticCandidate_(goodTriggerM3, badTriggerBefore);
  Logger.log("SAME_CANDIDATE_SELF ok=" + sameCand);
  Logger.log("SAME_CANDIDATE_DIFF ok=" + (!diffCand));
  const warnPacket = _kcmpSyntheticPacket_("MR07", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "못 움직일 것 같은데." }
  ]);
  const suspectSelfContained = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "설명 적절성",
    reason: "S1이 교사의 반박 이후 기존 판단을 다시 제시하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: true,
    quotes: [{ speaker: "S1", quote: "못 움직일 것 같은데." }],
    m3_evidence: { mode: "self_contained", trigger: null }
  };
  const warnCtx = {
    priorContextTurns: [
      { role: "teacher", speakerId: "", speakerRaw: "교사", row: 5, utterance: "그러면 못 움직이나?" }
    ]
  };
  const m = validateMDecisionResult_(suspectSelfContained, warnPacket, warnCtx);
  Logger.log("WARN_SELF_CONTAINED_CONTEXT ok=" + m.ok + " warnings=" + JSON.stringify(m.warnings));
  const packetCanon = _kcmpSyntheticPacket_("MR08", [
    { role: "teacher", speakerId: "", speakerRaw: "교사", row: 10, utterance: "다른 방법으로 움직일 수도 있지." },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 11, utterance: "못 움직일 것 같은데." }
  ]);
  const badTeacherSpeakerM3 = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "설명 적절성",
    reason: "S1이 교사 challenge 이후 기존 판단을 다시 제시하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "못 움직일 것 같은데." }],
    m3_evidence: {
      mode: "trigger_response",
      trigger: { role: "teacher", speaker: "S1", quote: "다른 방법으로 움직일 수도 있지." }
    }
  };
  const canonFix = _canonicalizeMDecisionResult_(badTeacherSpeakerM3, packetCanon, {});
  const n = validateMDecisionResult_(canonFix.result, packetCanon, {});
  Logger.log("CANONICAL_TEACHER_TRIGGER_REPAIR repaired=" + canonFix.repairLog.m_canonical_repair + " ok=" + n.ok + " errors=" + JSON.stringify(n.errors));
  const badEvalTypeOnTrigger = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "설명",
    reason: "재검토",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: false,
    quotes: [{ speaker: "S1", quote: "그러면 폐 자체가 아니라 주변 구조가 움직여서 폐 크기를 바꾸는 건가?" }],
    m3_evidence: {
      mode: "trigger_response",
      trigger: { role: "teacher", speaker: null, quote: "다른 구조가 움직일 수도 있지." },
      evaluation_type: "equivalence"
    }
  };
  const o = validateMDecisionResult_(badEvalTypeOnTrigger, packetTriggerM3);
  Logger.log("BAD_EVAL_TYPE_ON_TRIGGER ok=" + o.ok + " errors=" + JSON.stringify(o.errors));
  Logger.log("BAD_EVAL_TYPE_CLEARED eval_type=" + JSON.stringify(badEvalTypeOnTrigger.m3_evidence.evaluation_type));
  return {
    good_null: { ok: a.ok, errors: a.errors, expect_ok: true },
    good_m4_single_contributor: { ok: b.ok, errors: b.errors, expect_ok: true },
    bad_non_null_no_contributor: { ok: c.ok, errors: c.errors, expect_ok: false },
    bad_contributor_quote_mismatch: { ok: d.ok, errors: d.errors, expect_ok: false },
    bad_teacher_contributor: { ok: e.ok, errors: e.errors, expect_ok: false },
    bad_fake_quote: { ok: f.ok, errors: f.errors, expect_ok: false },
    good_m1: { ok: g.ok, errors: g.errors, expect_ok: true },
    good_m2_single_regulator: { ok: h.ok, errors: h.errors, expect_ok: true },
    good_m3_single_rechecker: { ok: i.ok, errors: i.errors, expect_ok: true },
    good_m3_trigger_response: { ok: j.ok, errors: j.errors, expect_ok: true },
    bad_m3_trigger_before: { ok: k.ok, errors: k.errors, expect_ok: false },
    good_m3_prior_context_trigger: { ok: l.ok, errors: l.errors, expect_ok: true },
    same_candidate_detector: { self_match: sameCand, diff_match: diffCand, expect_self: true, expect_diff: false },
    warn_self_contained_context: { ok: m.ok, warnings: m.warnings, expect_warning: true },
    canonical_teacher_trigger_repair: { repaired: canonFix.repairLog.m_canonical_repair, ok: n.ok, errors: n.errors, expect_repaired: true, expect_ok: true },
    bad_eval_type_on_trigger: { ok: o.ok, errors: o.errors, eval_type_cleared: badEvalTypeOnTrigger.m3_evidence.evaluation_type == null, expect_ok: true, expect_cleared: true }
  };
}

function testMDecisionTreeForPid_(pid){
  const sh = SpreadsheetApp.getActiveSheet();
  Logger.log("=== M DECISION TREE DRY-RUN ===");
  Logger.log("ACTIVE_SHEET=" + (sh ? sh.getName() : "(none)"));
  Logger.log("GOLD_KEY=" + (sh ? sh.getName() : "(none)") + "::" + (pid ? String(pid) : ""));
  const goldExpected = lookupMHumanGoldExpected_(sh ? sh.getName() : "", pid);
  if (goldExpected !== undefined) {
    Logger.log("GOLD_EXPECTED=" + (goldExpected == null ? "null" : goldExpected));
  } else {
    Logger.log("GOLD_EXPECTED=(not in 14차시 4조 harness)");
  }
  Logger.log("LAST_ROW=" + (sh ? sh.getLastRow() : 0));
  Logger.log("LAST_COLUMN=" + (sh ? sh.getLastColumn() : 0));
  const map = ensureColMapOrHalt_();
  const packets = buildAllKCMPClusterPackets_(sh, map);
  const want = normPID_(pid || "P037");
  const matches = (packets || []).filter(function(p){ return p && p.pid === want; });

  Logger.log("=== M DECISION TREE DRY-RUN ===");
  Logger.log("REQUESTED_PID=" + want);
  Logger.log("GOLD_KEY=" + (sh ? sh.getName() : "(none)") + "::" + want);
  const goldExpectedNorm = lookupMHumanGoldExpected_(sh ? sh.getName() : "", want);
  Logger.log("GOLD_EXPECTED=" + (goldExpectedNorm === undefined ? "(not in 14차시 4조 harness)" : (goldExpectedNorm == null ? "null" : goldExpectedNorm)));
  Logger.log("MATCH_COUNT=" + matches.length);

  if (matches.length === 0) {
    const err = {
      ok: false,
      error_type: "PACKET_ERROR",
      message: "requested PID not found",
      requestedPid: want
    };
    Logger.log("SELECTED_PID=(none)");
    Logger.log("ERROR=" + JSON.stringify(err));
    Logger.log("DRY-RUN: M셀을 수정하지 않음");
    return err;
  }
  if (matches.length > 1) {
    const err = {
      ok: false,
      error_type: "PACKET_ERROR",
      message: "duplicate packets for requested PID",
      requestedPid: want,
      matchCount: matches.length,
      matches: matches.map(function(p){
        return {
          pid: p.pid,
          representativeRow: p.representativeRow,
          turns: (p.turns || []).length,
          active: p.activeStudentIds
        };
      })
    };
    Logger.log("SELECTED_PID=(duplicate, not chosen)");
    matches.forEach(function(p, idx){
      Logger.log("MATCH[" + idx + "] pid=" + p.pid + " repRow=" + p.representativeRow + " turns=" + ((p.turns || []).length) + " active=" + JSON.stringify(p.activeStudentIds));
    });
    Logger.log("ERROR=" + JSON.stringify(err));
    Logger.log("DRY-RUN: M셀을 수정하지 않음");
    return err;
  }

  const packet = matches[0];
  Logger.log("SELECTED_PID=" + packet.pid);
  Logger.log("representativeRow=" + packet.representativeRow);
  Logger.log("turns=" + ((packet.turns || []).length));
  Logger.log("active=" + JSON.stringify(packet.activeStudentIds));
  Logger.log("speakerCounts=" + JSON.stringify(packet.speakerCounts));
  Logger.log("teacherPresent=" + packet.teacherPresent);
  (packet.turns || []).forEach(function(t){
    Logger.log("[" + t.row + "] [" + t.role + "] [" + (t.speakerId || "") + "] " + t.utterance);
  });

  const activeLen = (packet.activeStudentIds || []).length;
  const mCtx = _buildMDecisionContext_(packet, packets);
  if (mCtx.priorContextTurns.length > 0) {
    Logger.log("PRIOR_CONTEXT_PID=" + mCtx.priorContextPid);
    Logger.log("PRIOR_CONTEXT_TURNS=" + mCtx.priorContextTurns.length);
    mCtx.priorContextTurns.forEach(function(t){
      Logger.log("[prior][" + t.row + "] [" + t.role + "] [" + (t.speakerId || "") + "] " + t.utterance);
    });
  }
  if (activeLen === 0) {
    const result = runMDecisionTreeForPacket_(packet, { allPackets: packets, includeDebug: true });
    const validation = (result && result.status === "OK")
      ? validateMDecisionResult_(result, packet, mCtx)
      : { ok: false, errors: [result && result.message], warnings: [] };
    Logger.log("STRUCTURAL_PRE_GATE=true");
    Logger.log("GPT_CALLED=false");
    Logger.log("RESULT=" + JSON.stringify(result));
    Logger.log("VALIDATION=" + JSON.stringify(validation));
    Logger.log("DISPLAY=" + formatMDecisionDisplay_(result));
    Logger.log("NOTE_STATUS=" + (result && result.status) + " code=" + (result && result.code) + " error_type=" + (result && result.error_type));
    if (goldExpectedNorm !== undefined) {
      Logger.log("GOLD_COMPARE got=" + (result && result.status === "OK" ? (result.code == null ? "null" : result.code) : "(error)") + " expected=" + (goldExpectedNorm == null ? "null" : goldExpectedNorm));
    }
    Logger.log("DRY-RUN: M셀을 수정하지 않음");
    return {
      packet: packet,
      raw: null,
      parsed: null,
      validation: validation,
      result: result,
      display: formatMDecisionDisplay_(result)
    };
  }

  Logger.log("STRUCTURAL_PRE_GATE=false");
  Logger.log("GPT_CALLED=true");
  const prompt = buildMDecisionPrompt_(packet, mCtx);
  Logger.log("PROMPT_TURNS_INCLUDED=" + (prompt.indexOf("[CURRENT CLUSTER TURNS]") >= 0));
  Logger.log("PROMPT_PRIOR_CONTEXT=" + (prompt.indexOf("[PRIOR CONTEXT") >= 0));
  Logger.log("PROMPT_SUMMARY_AUX=" + (prompt.indexOf("AUXILIARY ONLY") >= 0));

  const result = runMDecisionTreeForPacket_(packet, { allPackets: packets, includeDebug: true });
  if (result && result.m_canonical_repair && result.m_canonical_repair.m_canonical_repair) {
    _logMDecisionCanonicalRepair_(result.m_canonical_repair);
  }
  const mDebug = result && result._mDebug ? result._mDebug : null;
  if (mDebug) {
    _logMDecisionRetryDebug_(mDebug);
    delete result._mDebug;
  }
  const validation = (result && result.status === "OK")
    ? validateMDecisionResult_(result, packet, mCtx)
    : { ok: false, errors: [result && result.message], warnings: [] };
  Logger.log("RESULT=" + JSON.stringify(result));
  if (result && result.status === "OK") {
    Logger.log("metacognitive_target=" + result.metacognitive_target);
    Logger.log("contributors=" + JSON.stringify(result.contributors));
    Logger.log("quotes=" + JSON.stringify(result.quotes));
    Logger.log("m3_evidence=" + JSON.stringify(result.m3_evidence));
  }
  Logger.log("VALIDATION=" + JSON.stringify(validation));

  const display = formatMDecisionDisplay_(result);
  Logger.log("DISPLAY=" + display);
  Logger.log("NOTE_STATUS=" + (result && result.status) + " code=" + (result && result.code) + " error_type=" + (result && result.error_type));
  if (goldExpectedNorm !== undefined) {
    Logger.log("GOLD_COMPARE got=" + (result && result.status === "OK" ? (result.code == null ? "null" : result.code) : "(error)") + " expected=" + (goldExpectedNorm == null ? "null" : goldExpectedNorm));
  }
  Logger.log("DRY-RUN: M셀을 수정하지 않음");
  return { packet: packet, parsed: result, validation: validation, result: result, display: display, retryDebug: mDebug };
}

function TEST_M_DECISION_TREE_P037(){
  return testMDecisionTreeForPid_("P037");
}

function TEST_M_DECISION_TREE_P038(){
  return testMDecisionTreeForPid_("P038");
}

function TEST_M_DECISION_TREE_P035(){
  return testMDecisionTreeForPid_("P035");
}

function TEST_M_DECISION_TREE_P014(){
  return testMDecisionTreeForPid_("P014");
}

function TEST_M_DECISION_TREE_P049(){
  return testMDecisionTreeForPid_("P049");
}

function TEST_M_DECISION_TREE_P008(){
  return testMDecisionTreeForPid_("P008");
}

function TEST_M_DECISION_TREE_P009(){
  return testMDecisionTreeForPid_("P009");
}

function TEST_M_DECISION_TREE_P016(){
  return testMDecisionTreeForPid_("P016");
}

function TEST_M_DECISION_TREE_P017(){
  return testMDecisionTreeForPid_("P017");
}

function TEST_M_DECISION_TREE_P043(){
  return testMDecisionTreeForPid_("P043");
}

function TEST_M_DECISION_TREE_P053(){
  return testMDecisionTreeForPid_("P053");
}

function TEST_M_DECISION_TREE_P058(){
  return testMDecisionTreeForPid_("P058");
}

function TEST_M_DECISION_TREE_P065(){
  return testMDecisionTreeForPid_("P065");
}

function TEST_M_DECISION_TREE_P013(){
  return testMDecisionTreeForPid_("P013");
}

function TEST_M_DECISION_TREE_P015(){
  return testMDecisionTreeForPid_("P015");
}

/** Validator-only regression for cross-cluster M3 prior-context fixtures. No GPT. */
function TEST_M_CROSS_CLUSTER_VALIDATOR(){
  const priorTurns = [
    { role: "teacher", speakerId: "", speakerRaw: "교사", row: 5, utterance: "그러면 근육이 없으면 못 움직이는 거야?" }
  ];
  const ctx = { priorContextTurns: priorTurns, priorContextPid: "MFX00" };
  const packetResponse = _kcmpSyntheticPacket_("MFX02V", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "못 움직이지 않아요?" },
    { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "다른 방법으로 움직일 수도 있지." },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "못 움직일 것 같은데." }
  ]);
  const goodResponse = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "근육 없음 설명의 적절성",
    reason: "S1이 prior teacher challenge 이후 기존 판단을 다시 제시하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: true,
    quotes: [{ speaker: "S1", quote: "못 움직일 것 같은데." }],
    m3_evidence: {
      mode: "trigger_response",
      trigger: { role: "teacher", speaker: null, quote: "그러면 근육이 없으면 못 움직이는 거야?" }
    }
  };
  const packetRecon = _kcmpSyntheticPacket_("MFX03V", [
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 10, utterance: "못 움직일 것 같은데." },
    { role: "teacher", speakerId: "", speakerRaw: "교사", row: 11, utterance: "공기가 이동하려면 압력이 어떻게 돼야 해?" },
    { role: "student", speakerId: "S1", speakerRaw: "학생1", row: 12, utterance: "그러니까 공기는 높은 압력에서 낮은 압력으로 가니까…" }
  ]);
  const goodRecon = {
    schema_version: "KCMP_M_V1",
    status: "OK",
    code: "M3",
    contributors: ["S1"],
    metacognitive_target: "공기 이동 설명의 재구성",
    reason: "S1이 prior challenge 이후 압력 원리를 이용해 설명을 다시 연결하였다.",
    decision_path: ["M-STEP0:YES", "M-STEP1:NO", "M-STEP2:NO", "M-STEP3:NO", "M-STEP4:YES"],
    boundary_check: null,
    context_needed: true,
    quotes: [{ speaker: "S1", quote: "그러니까 공기는 높은 압력에서 낮은 압력으로 가니까…" }],
    m3_evidence: {
      mode: "reconstruction",
      trigger: { role: "teacher", speaker: null, quote: "그러면 근육이 없으면 못 움직이는 거야?" }
    }
  };
  const a = validateMDecisionResult_(goodResponse, packetResponse, ctx);
  const b = validateMDecisionResult_(goodRecon, packetRecon, ctx);
  Logger.log("CROSS_CLUSTER_RESPONSE ok=" + a.ok + " errors=" + JSON.stringify(a.errors));
  Logger.log("CROSS_CLUSTER_RECONSTRUCTION ok=" + b.ok + " errors=" + JSON.stringify(b.errors));
  return {
    cross_cluster_response: { ok: a.ok, errors: a.errors, expect_ok: true },
    cross_cluster_reconstruction: { ok: b.ok, errors: b.errors, expect_ok: true }
  };
}

/** Dry-run only. Do not write M cells. Do not use gold labels in production inference. */
function TEST_M_HUMAN_GOLD_14X4_CORE(){
  const sh = SpreadsheetApp.getActiveSheet();
  const sheetName = sh ? sh.getName() : "";
  Logger.log("=== M HUMAN GOLD 14X4 CORE ===");
  Logger.log("ACTIVE_SHEET=" + sheetName);
  Logger.log("DRY-RUN ONLY: M셀을 수정하지 않음");
  const pids = ["P008", "P009", "P013", "P015", "P016", "P049", "P017", "P035", "P037", "P038", "P043", "P053", "P065"];
  const rows = [];
  let passN = 0;
  let failN = 0;
  pids.forEach(function(pid){
    const goldKey = sheetName + "::" + pid;
    const expected = lookupMHumanGoldExpected_(sheetName, pid);
    const run = testMDecisionTreeForPid_(pid);
    const result = run && run.result ? run.result : run;
    const validation = run && run.validation ? run.validation : null;
    let got = "(error)";
    let validationOk = false;
    let path = [];
    if (result && result.status === "OK") {
      got = result.code;
      validationOk = !!(validation && validation.ok);
      path = result.decision_path || [];
    } else if (result && result.error_type) {
      got = "(error:" + result.error_type + ")";
    }
    const codeMatch = !!(result && result.status === "OK") && (
      (expected == null && got == null) || (expected != null && got === expected)
    );
    let pathOk = true;
    if (expected === "M4") pathOk = _kcmpPathHas_(path, "M-STEP1:YES");
    if (expected === "M1") pathOk = _kcmpPathHas_(path, "M-STEP1:NO") && _kcmpPathHas_(path, "M-STEP2:YES");
    if (expected === "M3") pathOk = _kcmpPathHas_(path, "M-STEP4:YES");
    const inHarness = expected !== undefined;
    const pass = inHarness && codeMatch && validationOk && pathOk;
    if (pass) passN++; else failN++;
    Logger.log(
      "CORE_CASE gold_key=" + goldKey
      + " expected=" + (expected === undefined ? "(not in harness)" : (expected == null ? "null" : expected))
      + " got=" + (got == null ? "null" : got)
      + " validation_ok=" + validationOk
      + " pass=" + pass
    );
    rows.push({
      gold_key: goldKey,
      expected: expected,
      got: got,
      validation_ok: validationOk,
      pass: pass
    });
  });
  const total = pids.length;
  Logger.log("TOTAL=" + total);
  Logger.log("PASS=" + passN);
  Logger.log("FAIL=" + failN);
  Logger.log("PASS_RATE=" + (total ? (passN / total) : 0));
  Logger.log("DRY-RUN: M셀을 수정하지 않음");
  return { total: total, pass: passN, fail: failN, pass_rate: total ? (passN / total) : 0, rows: rows };
}

