## 들어가며

`Junit` 프레임워크에서 가져온 코드를 평가해보자.



## JUnit 프레임워크

살펴볼 모듈은 문자열 비교 오류를 파악할 때 유용한 코드다. `ComparisonCompactor` 라는 모듈로, 영리하게 짜인 코드다. `ComparisonCompactor` 는 두 문자열을 받아 차이를 반환한다. 예를 들어, `ABCDE` 와 `ABXDE` 를 받아 `(...B[X]D)` 를 반환한다.

```java
package junit.tests.framework;

import junit.framework.ComparisonCompactor;
import junit.framework.TestCase;

public class ComparisonCompactorTest extends TestCase {

    public void testMessage() {
        String failure = new ComparisonCompactor(0, "b", "c").compact("a");
        assertTrue("a expected:<[b]> but was:<[c]>".equals(failure));
    }

    public void testStartSame() {
        String failure = new ComparisonCompactor(1, "ba", "bc").compact(null);
        assertEquals("expected:<b[a]> but was:<b[c]>", failure);
    }

    public void testEndSame() {
        String failure = new ComparisonCompactor(1, "ab", "cb").compact(null);
        assertEquals("expected:<[a]b> but was:<[c]b>", failure);
    }

    public void testSame() {
        String failure = new ComparisonCompactor(1, "ab", "ab").compact(null);
        assertEquals("expected:<ab> but was:<ab>", failure);
    }

    public void testNoContextStartAndEndSame() {
        String failure = new ComparisonCompactor(0, "abc", "adc").compact(null);
        assertEquals("expected:<...[b]...> but was:<...[d]...>", failure);
    }

    public void testStartAndEndContext() {
        String failure = new ComparisonCompactor(1, "abc", "adc").compact(null);
        assertEquals("expected:<a[b]c> but was:<a[d]c>", failure);
    }

    public void testStartAndEndContextWithEllipses() {
        String failure = new ComparisonCompactor(1, "abcde", "abfde").compact(null);
        assertEquals("expected:<...b[c]d...> but was:<...b[f]d...>", failure);
    }

    public void testComparisonErrorStartSameComplete() {
        String failure = new ComparisonCompactor(2, "ab", "abc").compact(null);
        assertEquals("expected:<ab[]> but was:<ab[c]>", failure);
    }

    public void testComparisonErrorEndSameComplete() {
        String failure = new ComparisonCompactor(0, "bc", "abc").compact(null);
        assertEquals("expected:<[]...> but was:<[a]...>", failure);
    }

    public void testComparisonErrorEndSameCompleteContext() {
        String failure = new ComparisonCompactor(2, "bc", "abc").compact(null);
        assertEquals("expected:<[]bc> but was:<[a]bc>", failure);
    }

    public void testComparisonErrorOverlappingMatches() {
        String failure = new ComparisonCompactor(0, "abc", "abbc").compact(null);
        assertEquals("expected:<...[]...> but was:<...[b]...>", failure);
    }

    public void testComparisonErrorOverlappingMatchesContext() {
        String failure = new ComparisonCompactor(2, "abc", "abbc").compact(null);
        assertEquals("expected:<ab[]c> but was:<ab[b]c>", failure);
    }

    public void testComparisonErrorOverlappingMatches2() {
        String failure = new ComparisonCompactor(0, "abcdde", "abcde").compact(null);
        assertEquals("expected:<...[d]...> but was:<...[]...>", failure);
    }

    public void testComparisonErrorOverlappingMatches2Context() {
        String failure = new ComparisonCompactor(2, "abcdde", "abcde").compact(null);
        assertEquals("expected:<...cd[d]e> but was:<...cd[]e>", failure);
    }

    public void testComparisonErrorWithActualNull() {
        String failure = new ComparisonCompactor(0, "a", null).compact(null);
        assertEquals("expected:<a> but was:<null>", failure);
    }

    public void testComparisonErrorWithActualNullContext() {
        String failure = new ComparisonCompactor(2, "a", null).compact(null);
        assertEquals("expected:<a> but was:<null>", failure);
    }

    public void testComparisonErrorWithExpectedNull() {
        String failure = new ComparisonCompactor(0, null, "a").compact(null);
        assertEquals("expected:<null> but was:<a>", failure);
    }

    public void testComparisonErrorWithExpectedNullContext() {
        String failure = new ComparisonCompactor(2, null, "a").compact(null);
        assertEquals("expected:<null> but was:<a>", failure);
    }

    public void testBug609972() {
        String failure = new ComparisonCompactor(10, "S&P500", "0").compact(null);
        assertEquals("expected:<[S&P50]0> but was:<[]0>", failure);
    }
}
```

위 테스트 케이스로 `ComparisonCompactor` 모듈에 대한 코드 커버리지 분석을 수행했더니 100%가 나왔다. 테스트 케이스가 모든 `행`, 모든 `if`문, 모든 ` for`문을 실행한다는 의미다.

다음은 `ComparisonCompactor` 모듈이다. 코드는 잘 분리되었고, 표현력이 적절하며, 구조가 단순하다.

```java
/* ComparisonCompactor 원본 */

package junit.framework;

public class ComparisonCompactor {

    private static final String ELLIPSIS = "...";
    private static final String DELTA_END = "]";
    private static final String DELTA_START = "[";

    private int fContextLength;
    private String fExpected;
    private String fActual;
    private int fPrefix;
    private int fSuffix;

    public ComparisonCompactor(int contextLength, String expected, String actual) {
        fContextLength = contextLength;
        fExpected = expected;
        fActual = actual;
    }

    @SuppressWarnings("deprecation")
    public String compact(String message) {
        if (fExpected == null || fActual == null || areStringsEqual()) {
            return Assert.format(message, fExpected, fActual);
        }

        findCommonPrefix();
        findCommonSuffix();
        String expected = compactString(fExpected);
        String actual = compactString(fActual);
        return Assert.format(message, expected, actual);
    }

    private String compactString(String source) {
        String result = DELTA_START + source.substring(fPrefix, source.length() - fSuffix + 1) + DELTA_END;
        if (fPrefix > 0) {
            result = computeCommonPrefix() + result;
        }
        if (fSuffix > 0) {
            result = result + computeCommonSuffix();
        }
        return result;
    }

    private void findCommonPrefix() {
        fPrefix = 0;
        int end = Math.min(fExpected.length(), fActual.length());
        for (; fPrefix < end; fPrefix++) {
            if (fExpected.charAt(fPrefix) != fActual.charAt(fPrefix)) {
                break;
            }
        }
    }

    private void findCommonSuffix() {
        int expectedSuffix = fExpected.length() - 1;
        int actualSuffix = fActual.length() - 1;
        for (; actualSuffix >= fPrefix && expectedSuffix >= fPrefix; actualSuffix--, expectedSuffix--) {
            if (fExpected.charAt(expectedSuffix) != fActual.charAt(actualSuffix)) {
                break;
            }
        }
        fSuffix = fExpected.length() - expectedSuffix;
    }

    private String computeCommonPrefix() {
        return (fPrefix > fContextLength ? ELLIPSIS : "") + fExpected.substring(Math.max(0, fPrefix - fContextLength), fPrefix);
    }

    private String computeCommonSuffix() {
        int end = Math.min(fExpected.length() - fSuffix + 1 + fContextLength, fExpected.length());
        return fExpected.substring(fExpected.length() - fSuffix + 1, end) + (fExpected.length() - fSuffix + 1 < fExpected.length() - fContextLength ? ELLIPSIS : "");
    }

    private boolean areStringsEqual() {
        return fExpected.equals(fActual);
    }
}
```

긴 표현식 몇 개와 이상한 +1 등이 눈에 띈다. 하지만 전반적으로 상당히 훌륭한 모듈이다. 자칫하면 아래처럼 작성했을 수도 있으니까.

```java
/* 디팩터링 결과 */

package junit.framework;

public class ComparisonCompactor {
  private static final String ELLIPSIS = "...";
  private static final String DELTA_END = "]";
  private static final String DELTA_START = "[";
  
  public ComparisonCompactor(int contextLength, String expected, String actual) {
        fContextLength = contextLength;
        fExpected = expected;
        fActual = actual;
  }
  
  public String compact(String msg) {
    if (s1 == null || s2 == null || s1.equals(s2))
      return Assert.format(msg, s1, s2);
    
    pfx = 0;
    for (; pfx < Math.min(s1.length(), s2.length()); pfx++) {
      if(s1.charAt(pfx) != s2.charAt(pfx)) {
        break;
      }
    }
    int sfx1 = s1.length() - 1;
    int sfx2 = s2.length() - 1;
    for (; sfx >= pfx && sfx1 >= pfx; sfx2--, sfx1--) {
      if (s1.charAt(sfx1) != s2.charAt(sfx2)) {
        break;
      }
    }
    sfx = s1.length() - sfx1;
    String cmp1 = compactString(s1);
    String cmp2 = compactString(s2);
    return Assert.format(msg, cmp1, cmp2);
  }
  
  private String compactString(String s) {
    String result = "[" + s.substring(pfx, s.length() - sfx + 1) + "]";
    if(pfx > 0) {
      result = (pfx > ctxt ? "..." : "") + 
        s1.substring(Math.max(0, pfx - ctxt), pfx) + result;
    }
    if (sfx > 0) {
      int end = Math.min(s1.length() - sfx + 1 + ctxt, s1.length());
      result = result + (s1.substring(s1.length() - sfx + 1, end) + 
                         (s1.length() - sfx + 1 < s1.length() - ctxt ? "..." : ""));
    }
  	return result;
  } 
}
```

비록 저자들이 모듈을 아주 좋은 상태로 남겨두었지만 `보이스카우트 규칙` 에 따르면 우리는 처음 왔을 때보다 더 깨끗하게 놓고 떠나야 한다. 그렇다면 `ComparisonCompactor` 원본을 어떻게 개선하면 좋을까?

가장 눈에 거슬리는 부분은 멤버 변수 앞에 붙인 접두어다.

```java
/* Bad */
private int fContextLength;
private String fExpected;
private String fActual;
private int fPrefix;
private int fSuffix;

/* Good */
private int contextLength;
private String expected;
private String actual;
private int prefix;
private int suffix;
```

다음으로  `compact` 함수 시작부에 캡슐화되지 않은 조건문이 보인다.

```java
/* Bad */
public String compact(String message) {
        if (expected == null || actual == null || areStringsEqual()) { // 이 부분
            return Assert.format(message, expected, actual);
        }

        findCommonPrefix();
        findCommonSuffix();
        String expected = compactString(this.expected);
        String actual = compactString(this.actual);
        return Assert.format(message, expected, actual);
}
```

의도를 명확히 표현하려면 `조건문` 을 캡슐화해야 한다. 즉, 조건문을 메서드로 뽑아내 적절한 이름을 붙인다.

```java
public String compact(String message) {
        if (shouldNotCompact()) {
            return Assert.format(message, expected, actual);
        }

        findCommonPrefix();
        findCommonSuffix();
        String expected = compactString(this.expected);
        String actual = compactString(this.actual);
        return Assert.format(message, expected, actual);
  
  			private boolean shouldNotCompact() {
          return expected == null || actual == null || areStringsEqual();
        }
}
```

`compact` 함수에서 사용하는 `this.expected` 와 `this.actual` 도 눈에 거슬린다. (함수에 이미 `expected` 라는 지역 변수가 있는데) `fExpected` 에서 `f` 를 빼버리는 바람에 생긴 결과다. 함수에서 멤버 변수와 이름이 똑같은 변수를 사용하는 이유가 무엇일까? 서로 다른 의미가 아닌가? 이름은 명확하게 붙인다.

```java
/* Bad */
String expected = compactString(this.expected);
String actual = compactString(this.actual);

/* Good */
String compactExpected = compactString(expected);
String compactActual = compactString(actual);
```

부정문은 긍정문보다 이해하기 약간 더 어렵다. 그러므로 첫 문장 `if` 를 긍정으로 만들어 조건문을 반전한다.

```java
public String compact(String message) {
        if (canBeCompacted()) {
        	findCommonPrefix();
        	findCommonSuffix();
        	String compactExpected = compactString(expected);
					String compactActual = compactString(actual);
        	return Assert.format(message, compactExpected, compactActual);
        }
  			else {
        	return Assert.format(message, expected, actual);
        }

  			private boolean canBeCompacted() {
          return expected != null && actual != null && !areStringsEqual();
        }
}
```

함수 이름이 이상하다. 문자열을 압축하는 함수라지만 실제로 `canBeCompacted` 가 `false` 면 압축하지 않는다. (`Can Be Compacted? = True` 문맥상 이렇게 체크하는게 어색하지 않은데 말이다.) 그러므로 함수에 `compact` 라는 이름을 붙이면 `오류 점검` 이라는 부가 단계가 숨겨진다. 게다가 함수는 단순히 압축된 문자열이 아니라 형식이 갖춰진 문자열을 반환한다. 따라서 실제로는 `formatCompactedComparison` 이라는 이름이 적합하다.

```java
public String formatCompactedComparison(String message) { ... }
```

`if` 문 안에서는 예상 문자열과 실제 문자열을 진짜로 압축한다.

```java
...

private String compactExpected;
private String compactActual;

...

public String compact(String message) {
        if (canBeCompacted()) {
        	compactExpectedAndActual()
        	return Assert.format(message, compactExpected, compactActual);
        }
  			else {
        	return Assert.format(message, expected, actual);
        }

  			private void compactExpectedAndActual() {
          findCommonPrefix();
        	findCommonSuffix();
        	String compactExpected = compactString(expected);
					String compactActual = compactString(actual);
        }
}
```

새 함수에서 마지막 두 줄은 변수를 반환하지만 첫째 줄과 둘째 줄은 반환값이 없다. 함수 사용방식이 일관적이지 못하다. 그래서 `findCommonPrefix` 와 `findCommonSuffix` 를 변경해 접두어 값과 접미어 값을 반환한다.

```java
private void compactExpectedAndActual() {
	prefixIndex = findCommonPrefix();
	suffixIndex = findCommonSuffix();
	String compactExpected = compactString(expected);
	String compactActual = compactString(actual);
}

private iont findCommonPrefix() {
  int prefixIndex = 0;
  int end = Math.min(expected.length(), actual.length());
  for (; prefixIndex < end; prefixIndex++) {
    if (expected.charAt(prefixIndex) != actual.charAt(prefixIndex)) {
      break;
    }
  }
  return prefixIndex;
}

private int findCommonSuffix() {
  int expectedSuffix = expected.length() - 1;
  int actualSuffix = actual.length() - 1;
  for (; actualSuffix >= prefixIndex && expectedSuffix >= prefixIndex; 
       actualSuffix--, expectedSuffix--) {
    if (expected.charAt(expectedSuffix) != actual.charAt(actualSuffix)) {
      break;
    }
  }
  return expected.length() - expectedSuffix;
}
```

`findCommonSuffix` 를 주의깊게 살펴보면 `숨겨진 시간적인 결함` 이 존재한다. 다시말해 `findCommandSuffix` 는 `findCommonPrefix` 가 `prefixIndex` 를 계산한다는 사실에 의존한다. 만약 `findCommonPrefix` 와 `findCommonSuffix` 를 잘못된 순서로 호출하면 밤샘 디버깅이라는 고생문이 열린다. 그래서 시간 결함을 외부에 노출하고자 `findCommonSuffix` 를 고쳐 `prefixIndex` 를 인수로 넘겼다.

```java
private void compactExpectedAndActual() {
	prefixIndex = findCommonPrefix();
	suffixIndex = findCommonSuffix(prefixIndex);
	String compactExpected = compactString(expected);
	String compactActual = compactString(actual);
}

private int findCommonSuffix(int prefixIndex) {
  int expectedSuffix = expected.length() - 1;
  int actualSuffix = actual.length() - 1;
  for (; actualSuffix >= prefixIndex && expectedSuffix >= prefixIndex; 
       actualSuffix--, expectedSuffix--) {
    if (expected.charAt(expectedSuffix) != actual.charAt(actualSuffix)) {
      break;
    }
  }
  return expected.length() - expectedSuffix;
}
```

사실 이 방법이 썩 내키지는 않는다. 함수 호출 순서는 확실히 정해지지만 `prefixIndex` 가 필요한 이유는 설명하지 못한다. 그러므로 이번에는 다른 방식을 고안해보자.

```java
private void compactExpectedAndActual() {
	findCommonPrefixAndSuffix();
	String compactExpected = compactString(expected);
	String compactActual = compactString(actual);
}

private void findCommonPrefixAndSuffix() {
  findCommonPrefix();
  int expectedSuffix = expected.length() - 1;
  int actualSuffix = actual.length() - 1;
  for (; actualSuffix >= prefixIndex && expectedSuffix >= prefixIndex; 
       actualSuffix--, expectedSuffix--) {
    if (expected.charAt(expectedSuffix) != actual.charAt(actualSuffix)) {
      break;
    }
  }
  return expected.length() - expectedSuffix;
}
```

이렇게 수정할 경우 `findCommonPrefix` `findCommonSuffix` 함수를 호철하는 순서가 훨씬 더 분명해진다. 또한 `findCommonPrefixAndSuffix` 함수가 얼마나 지저분한지도 드러난다. 이제 함수를 정리해보자.

```java
private void findCommonPrefixAndSuffix() {
  findCommonPrefix();
  int suffixLength = 1;
  for (; !suffixOverlapsPrefix(suffixLength); suffixLength++) {
    if (charFromEnd(expected, suffixLength) != charFromEnd(actual, suffixLength)) {
      break;
    }
  }
  suffixIndex = suffixLength;
}

private char charFromEnd(String s, int i) {
  return s.charAt(s.length() - i);
}

private boolean suffixOverlapsPrefix(int suffixLength) {
  return actual.length() - suffixLength < prefixLength || expected.length() - suffixLength < prefixLength;
}
```

고치고 나니 `suffixIndex` 가 실제로는 접미어 길이라는 사실이 드러난다. 이름이 적절하지 못하다는 의미이다. `prefixIndex` 도 마찬가지로, 이 경우 `index` 와 `length` 가 동의어다. 비록 그렇다 하더라도, `length` 가 더 합당하다. 실제로 `suffixIndex` 는 0에서 시작하지 않는다. 1에서 시작하므로 진정한 길이가 아니다. `computeCommonSuffix` 에 `+1` 이 곳곳에 등장하는 이유가 여기에 있다. 고쳐보자.

```java
public class ComparisonCompactor {
  ...
  private int suffixLength; // suffixIndex ➜ suffixLength
  ...
	private void findCommonPrefixAndSuffix() {
    findCommonPrefix();
    suffixLength = 0;
    for (; !suffixOverlapsPrefix(suffixLength); suffixLength++) {
      if (charFromEnd(expected, suffixLength) != charFromEnd(actual, suffixLength)) {
        break;
      }
    }
    suffixIndex = suffixLength;
  } 
  
  private char charFromEnd(String s, int i) {
    return s.charAt(s.length() - i - 1);
  }
  
  private boolean suffixOverlapsPrefix(int suffixLength) {
    return actual.length() - suffixLength < prefixLength 
      || expected.length() - suffixLength < prefixLength;
  }
  
  ...
    
  private String compactString(String source) {
		String result = DELTA_START + source.substring(prefixLength, source.length() - suffixLength) + DELTA_END;
		if (prefixLength > 0) {
			result = computeCommonPrefix() + result;
		}
		if (suffixLength > 0) {
			result = result + computeCommonSuffix();
		}
		return result;
	}
  
  ...
    
	private String computeCommonSuffix() {
    int end = Math.min(fExpected.length() - suffixLength + fContextLength, fExpected.length());
		return fExpected.substring(fExpected.length() - suffixLength, end) + (fExpected.length() - suffixLength < fExpected.length() - fContextLength ? ELLIPSIS : "");
  }
}
```

`+1` 연산을 없애고 `suffixIndex` 를 `suffixLength` 로 변경했다. 이로써 코드 가독성이 크게 높아졌다.

그런데 문제가 하나 생겼다. `+1` 을 제거하던 중 `compactString` 에서 다음 행을 발견했다.

```java
if (suffixLength > 0) 
```

`suffixLength` 가 이제 1씩 감소했으므로 당연히 `>` 연산자를 `>=` 연산자로 고쳐야 마땅하다. 하지만 `>=` 연산자는 말이 안 된다! 지금 그대로 `>` 연산자가 맞다! 즉, 원래 코드가 틀렸으며 필경 버그라는 말이다. 아니, 엄밀하게 버그는 아니다. 원래 코드는 `suffixIndex` 가 언제나 1 이상이었으므로 `if` 문 자체가 있으나 마나였다.

그러고 보니 `compactString` 에 있는 `if` 문 둘 다 의심스럽다. 둘 다 필요 없어보인다.

```java
  private String compactString(String source) {
		return result = computeCommonPrefix() + 
      							DELTA_START + 
      							source.substring(prefixLength, source.length() - suffixLength) + 
      							DELTA_END +
      							computeCommonSuffix();
	}
```

이제 `compactString` 함수는 단순히 문자열 조각만 결합한다. 좀 더 깔끔하게 정리할 여지는 존재한다.



## 결론

저자들은 우수한 모듈을 만들었다. 하지만 세상에 개선이 불필요한 모듈은 없다. 코드를 처음보다 조금 더 깨끗하게 만드는 책임은 우리 모두에게 있다.



## 생각 정리

* 거대한 프레임워크라 해서 한 프로그램 파일의 크기가 클 줄 알았으나, 생각보다 모듈화가 잘 되어있어 하나의 파일 속 코드 내용은 적었다.
* 많은 개발자들이 사용하고 있는 코드 조차도 내부를 들여다보고 구조를 파악함으로써, 더 깨끗한 코드로의 개선의 여지를 찾을 수 있다는 것을 깨달았다.
* 리팩터링 과정 중간에서 멤버 변수를 멤버 함수 내의 매개변수로 넘겨주는 예제를 보았다. 멤버 변수를 굳이? 저렇게 넘겨야 하나 라는 생각이 들었지만, 이러한 방법이 코드의 실행 순서를 명확하게 표현해줌으로써 시간의 결함을 해결해 줄 수도 있다는 것을 알게됐다. `하지만 책에서도 결국 또 리팩터링 했듯이, 이런 방법은 사용하지 말자!` 정확한 순서로 실행될 수 있도록 함수를 추가하자.
* 변수명을 가독성 좋게 바꾸고, 함수의 추상화 수준을 동일하게 유지하는 과정을 보며, 크게 뜯어 고치는 작업이 아니지만 이런 노력으로도 코드가 처음보다 확연히 깨끗해 진다는 사실을 확인할 수 있었다.
* 리팩터링을 하다 보면 사실은 필요 없었던, 없었어도 테스트 케이스를 통과하는데 문제 없는 로직을 찾아낼 수 있고 이를 제거함으로써 간결한 코드를 작성할 수 있다는 것을 알게 됐다. 