# LLM Council-korean

![llm-council-korean](llm-council-korean.png)

이 repo의 아이디어는, 선호하는 LLM provider(예: OpenAI GPT 5.1, Google Gemini 3.0 Pro, Anthropic Claude Sonnet 4.5, xAI Grok 4 등)에게 질문을 하는 대신, 그들을 하나로 묶어 당신의 “LLM Council-korean”로 만드는 것입니다. 이 repo는 단순하고 로컬에서 실행되는 web app인데 본질적으로 ChatGPT처럼 보이지만, OpenRouter를 사용해서 당신의 질의(query)를 여러 LLM에게 보내고 그 다음에 서로의 작업물을 리뷰하고 순위 매기기(ranking)를 하도록 요청하며 마지막에는 의장(Chairman) LLM이 최종 응답(final response)을 생성합니다.

조금 더 자세히 말하자면, 질의(query)를 제출했을 때 다음과 같은 과정이 일어납니다:

1. **Stage 1: 첫 의견(First opinions)**. 사용자 질의(user query)가 각 LLM에게 개별적으로 전달되고, responses가 수집됩니다. 이 개별 응답(responses)은 “tab view”에 표시되어, 사용자가 각 LLM의 응답을 하나씩 확인할 수 있습니다.

2. **Stage 2: 검토(Review)**. 각 개별 LLM에게 다른 LLM들의 응답(responses)이 제공됩니다. 내부적으로 LLM 정체성(identity)은 익명화되어 평가시 특정 결과값(output)에 편향이 생기지 않도록 합니다. 각 LLM은 정확도(accuracy)와 통찰(insight)을 기준으로 순위 매기기(ranking)를 하도록 요청받습니다. 

3. **Stage 3: 최종 응답(Final response)**. LLM Council-korean의 지정된 의장(Chairman)이 모든 모델의 응답(responses)을 받아 하나의 최종 정답(answer)으로 정리하고 이것이 사용자에게 제공됩니다..

## Vibe Code Alert

이 프로젝트는 Karpathy거의 99% vibe coding으로 진행된 토요일 해킹 프로젝트(Saturday hack)으로써 여러 LLM들을 나란히 비교·평가하기 위해 “LLM들과 함께 책 읽기[reading books together with LLMs](https://x.com/karpathy/status/1990577951671509438)" 과정의 일부였습니다. 저 또한 여러 응답(responses)을 나란히 보는 것은 좋고 유용하며, 각 LLM이 다른 LLM outputs에 대해 가지는 cross-opinion을 보는 것 역시 흥미로웠습니다. 그러다가 마침 Karpathy의 "나는 이 프로젝트를 어떤 방식으로도 지원할 생각이 없고, 다른 사람들의 영감을 위해 as-is 상태로 제공한다. 개선할 의도도 없다"에서 이 프로젝트의 개선선이 기대되었던 것에 대해서 느낀 아쉬움을 직접  cross-opinion을 통해 vibe coding 목-토 프로젝트(Thursday to Saturday, Thur2sat)로 원하는 방식으로 바꾸면 어떨까 하였습니다.

아마도 저는 "지금 시대에 코드는 일시적이며, 라이브러리 시대는 끝났고, 원하는 방식으로 바꾸는 것은 너의 LLM에게 요청하라."를 충실히 이행하고 있다고 충분히 느끼고 있습니다. 
