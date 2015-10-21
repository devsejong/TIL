# intelliJ 단축키

## cheatsheat

pdf버전 프린트해서 두고두고 보도록 하자.

- 윈도 : https://www.jetbrains.com/idea/docs/IntelliJIDEA_ReferenceCard.pdf
- 맥 : https://www.jetbrains.com/idea/docs/IntelliJIDEA_ReferenceCard_Mac.pdf

## 유용한 단축키들 

- `command + shift + a` : intelliJ의 모든 액션을 찾아볼 수 있다.(읜도의 경우 `ctrl + shift + a`)
- `ctrl + shift + alt + v` : intelliJ에서 문자열을 붙여넣을 때, 자동으로 포메팅을 진행한다. 위의 단축키를 사용할 경우 포메팅을 하지 않고 바로 붙여넣기가 가능하다. 
 
### navigation

- `Ctrl+B` : 커서가 위치한 곳의 메서드를 선언한 인터페이스로 이동한다.
- `Ctrl+Alt+B` : 커서가 위치한 곳의 메서드를 구현한 클래스로 이동한다.
- `Ctrl+U` : 커서가 위치한 곳 메서드의 super class로 이동한다.

## 오류별 해결방법

(윈도)SVN이 설치되어 있으나 SVN을 실행할 때 다음과 같은 오류가 발생하는 경우

```
Cannot load supported formats: Cannot run program "svn": CreateProcess error=2, The system cannot find the file specified
```

설정의 [Version Control] - [Subversion]에서 Use command line client를 체크해제한다.
