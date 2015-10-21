# jQuery chained

- http://www.appelsiini.net/projects/chained

## 개요



jquery.chained.js는 여러개의 selectbox가 계층형 구조를 가질 경우, 즉 첫번째 selectbox의 선택값에 따라서 다음 selectbox의 선택가능한 항목만 보여주어야 하는 경우에 사용할 수 있는 라이브러리이다.

## 사용법


첫번째 selectbox에서 선택가능한 값 bmw, audi을 선택하면, 두번째 selectbox의 class의 값 중 선택한 값과 일치하는 클래스만을 표시한다.

```
<select id="mark" name="mark">
  <option value="">--</option>
  <option value="bmw">BMW</option>
  <option value="audi">Audi</option>
</select>
<select id="series" name="series">
  <option value="">--</option>
  <option value="series-3" class="bmw">3 series</option>
  <option value="series-5" class="bmw">5 series</option>
  <option value="series-6" class="bmw">6 series</option>
  <option value="a3" class="audi">A3</option>
  <option value="a4" class="audi">A4</option>
  <option value="a5" class="audi">A5</option>
</select>
```

스크립트와 라이브러리 추가는 다음과 같이 할 수 있다.

```
<script src="//ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
<script src="jquery.chained.min.js"></script>
<script>
$("#series").chained("#mark"); /* or $("#series").chainedTo("#mark"); */
</script>
```

![2015-10-21 11 38 31](https://cloud.githubusercontent.com/assets/3116341/10639774/dea60b64-784c-11e5-81cd-079e6119b73b.png)

