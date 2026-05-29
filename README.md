# manuscript-site-json

[Manuscript](https://zendy00.github.io/manuscript-pages/) 홈페이지를 위한 **가이드 투어 시나리오 JSON** 저장소입니다.

Manuscript는 웹 페이지를 단계별 매뉴얼로 바꿔주는 브라우저 확장입니다. 이 저장소의 JSON 파일들은 Manuscript 런타임이 재생하는 시나리오(스포트라이트 + 내레이션 + 주석)를 정의하며, 모두 `schemaVersion 0.1.2`를 따릅니다.

## 구성

각 시나리오는 영어(`-en`)와 한국어(`-ko`) 두 언어로 제공됩니다. 두 언어 버전은 동일한 셀렉터를 공유하고 내레이션 텍스트만 다릅니다.

| 시나리오 | 대상 페이지 | 파일 | 스텝 |
| --- | --- | --- | --- |
| 랜딩 투어 | 메인 랜딩 페이지 | `tour-en.json`, `tour-ko.json` | 7 |
| 개인정보 처리방침 | Privacy 페이지 | `privacy-en.json`, `privacy-ko.json` | 6 |
| 변경 이력 | Changelog 페이지 | `changelog-en.json`, `changelog-ko.json` | 4 |

- **`SKILL.md`** — 위 시나리오 JSON을 작성·번역·확장하는 방법을 담은 에이전트 스킬 문서입니다. AI 에이전트가 페이지를 분석해 셀렉터를 고르고 내레이션을 작성한 뒤 JSON을 생성하는 절차, 스키마 레퍼런스, 검증 체크리스트가 포함되어 있습니다.

## JSON 스키마 개요

각 시나리오 파일은 다음 형태를 가집니다.

```json
{
  "schemaVersion": "0.1.2",
  "id": "tour-manuscript-landing-en",
  "name": "Manuscript landing page walkthrough",
  "createdAt": "2026-05-23T13:00:00Z",
  "siteFonts": ["..."],
  "steps": [
    {
      "id": "step-1-hero",
      "name": "What Manuscript Is",
      "description": "내레이션 텍스트",
      "selectors": { "...": "..." },
      "annotations": [],
      "autoAdvanceMs": 9500
    }
  ]
}
```

자세한 스키마와 작성 규칙은 [`SKILL.md`](./SKILL.md)를 참고하세요.

## 사용법

1. 원하는 시나리오 JSON 파일을 Manuscript 런타임(브라우저 확장 또는 런타임 브리지)에 전달합니다.
2. 재생하면 해당 페이지에서 스포트라이트와 내레이션 투어가 진행됩니다.

새 시나리오를 만들거나 기존 시나리오를 수정·번역할 때는 `SKILL.md`의 절차를 따릅니다.
