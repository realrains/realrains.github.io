---
layout: post
title: 테스트 코드에서 ktlint 규칙 비활성화 하기
date: 2024-01-13
---

[ktlint](https://pinterest.github.io/ktlint/latest) 는 코틀린 코드에 대한 정적 분석 도구입니다. 코드 스타일 가이드라인을 프로젝트 코드 베이스에 대해 자동으로 적용하여 일관성 있는 코드 스타일을 유지하는 데 도움을 줄 수 있습니다.

진행하고 있는 프로젝트에서는 gradle 플러그인 [ktlint-gradle](https://github.com/JLLeitschuh/ktlint-gradle) 을 적용해 빌드 시 코드 스타일 가이드를 위반한 사례가 있는지 자동으로 검증하고 있는데요, 이 경우 프로젝트 루트의 `.editorconfig` 파일로 적용할 규칙을 다음과 같이 커스터마이징 할 수 있습니다.

```toml
root = true

[*]
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.{kt,kts}]
lj_kotlin_code_style_defaults = KOTLIN_OFFICIAL
```

다만 테스트 코드를 작성할 때에는 프로덕션 코드와 다른 규칙이 적용되어야 할 필요가 있을 수 있습니다. 대표적으로 코틀린에서 테스트 메서드를 작성할 때 메서드 이름을 백틱(`` ` ``)으로 감싸 표현하는 사례가 이에 해당합니다.

{% include image.html url='/assets/image/naming-test-function.png' %}

개인적으로는 테스트 메서드는 한글로 표현하는 것이 요구사항을 잘 검증하고 있는지 파악하는 데 용이하다고 생각하는데요, 자바의 경우 메서드 명에 한글을 사용할 수는 있지만 띄어쓰기를 쓸 수 없기 때문에 `1+1_의_결과는_2_이다` 와 같이 언더바를 포함하여 쓰거나 테스트 메서드명 자체는 영어로 짓고 `@DisplayName` 을 사용해 원하는 테스트 케이스 명을 짓곤 했습니다.

일반적인 메서드 명과 달리 테스트 메서드는 서술적인 면이 더 부각되기 때문에 영어로 테스트 명을 짓는 것이 쉬운 일이 아니거니와 영어로 테스트 메서드명을 짓는다고 해도 표준 스타일 가이드 대로 camel case 로 쓰게되면 긴 문장의 경우 가독성이 현저히 떨어지게 됩니다. 그렇다고 영문 메서드 명을 대강 짓고 `@DisplayName` 을 사용하는 것도 조금 불편한 면이 있는것도 사실입니다.

다행히 코틀린의 경우는 앞서 말한 대로 백틱(`` ` ``)으로 감싸 일반적인 문장의 형태로 메서드 명을 짓는 것이 가능한데요 하지만 이 경우 ktlint 의 표준 스타일 규칙 중 하나인 function-naming 을 위반하게 됩니다. 다행히 ktlint 에서는 테스트 코드에 한해 일부 규칙을 적용하지 않고 있는데요 `kotest`, `junit` 등 테스트 관련 패키지가 임포트 된 파일의 경우 이를 무시해 줍니다.

```kotlin
// 허용
fun foo() {}
fun fooBar() {}
fun `fun` {}

// 허용되지 않음
fun Foo() {}
fun Foo_Bar() {}
fun `Some name`() {}
fun do_something() {}

// 테스트 코드에서만 허용
@Test
fun `Some name`() {}
@Test
fun do_something() {}
```

그러나 테스트 코드를 작성하다 보면 테스트 작성에 필요한 픽스처, 재사용을 위한 테스트 스텝 등 헬퍼 코드가 필요할 때가 있는데요 이 경우 ktlint 의 테스트 코드 예외가 적용되지 않게 됩니다.

{% include image.html url='/assets/image/test-fixture-function.png' %}

이 경우 다음과 같이 테스트 코드가 포함된 디렉터리에 대해 ktlint 룰을 오버라이딩 할 수 있습니다.

```toml
[**/src/test/**.{kt,kts}]
ktlint_standard_function-naming = disabled
```

## References

* [ktlint - Sandard rules](https://pinterest.github.io/ktlint/latest/rules/standard)
* [ktlint - Overriding properties for specific directories](https://pinterest.github.io/ktlint/latest/rules/configuration-ktlint/#overriding-editorconfig-properties-for-specific-directories)