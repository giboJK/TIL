

> 2026-08-24 · 5장 정리 손실 함수가 왜 필요한지부터, 문제 유형별 loss 선택, optimizer와 learning rate, 그리고 `zero_grad → forward → backward → step` 한 사이클까지. 강의를 들으며 스스로 물어봤던 질문과 답을 정리한다.

**전체 그림**

```
loss: "얼마나 틀렸나" → 숫자 하나
gradient: "어느 방향으로 가면 loss가 커지나" → 방향
optimizer: "그 반대로 얼마나 갈까" → 실제 parameter 변경
```

세 역할이 분리되어 있다는 것이 5장의 핵심이다.

---

## 1. 손실 함수는 왜 필요한가

**Q. 정답/오답만 세면 안 되나?**

맞았다/틀렸다는 0 아니면 1이라 **미분이 안 된다.** 경사하강법은 기울기로 방향을 찾는데, 계단 함수에는 기울기가 없다.

그리고 정확도는 _얼마나_ 틀렸는지를 버린다. 정답이 10인데 9로 예측한 것과 100으로 예측한 것이 똑같이 "오답 1개"가 된다. 손실 함수는 이 크기를 보존한다.

**Q. loss function과 objective function은 같은 말인가?**

거의 같이 쓰지만 범위가 다르다.

- **objective function** — 최적화 대상 전체. 최소화일 수도 최대화일 수도 있다
- **loss function** — 보통 최소화 대상. objective의 한 부분

정규화가 붙으면 `objective = loss + λ·penalty`가 되어 둘이 달라진다.

**Q. prediction, target, loss의 관계는?**

```
loss = criterion(prediction, target)
```

- `prediction` — **gradient를 가진다** (계산 그래프에 연결)
- `target` — 정답, 상수 취급
- `loss` — 스칼라 하나. 여기서 `backward()`가 출발한다

**Q. MSELoss를 직접 계산하면?**

```python
pred   = torch.tensor([2.0, 4.0, 6.0])
target = torch.tensor([3.0, 4.0, 8.0])

manual = ((pred - target) ** 2).mean()   # (1 + 0 + 4) / 3 = 1.6667
built  = nn.MSELoss()(pred, target)      # 1.6667
```

**Q. reduction 옵션은?**

샘플별 loss를 **배치 하나의 숫자로 어떻게 합칠지** 정한다.

|값|결과|언제|
|---|---|---|
|`'mean'` (기본)|평균|대부분|
|`'sum'`|합|배치 크기에 비례해 gradient가 커짐|
|`'none'`|샘플별 loss 유지|가중치·마스킹 등 후처리|

기본이 `mean`인 이유는 **배치 크기가 바뀌어도 loss 스케일이 일정하기 때문**이다. `sum`을 쓰면 배치를 32→64로 늘리는 순간 gradient가 두 배가 되어 lr을 다시 잡아야 한다.

```python
crit = nn.MSELoss(reduction='none')
per_sample = crit(pred, target)
loss = (per_sample * weight).mean()   # 샘플 가중치
loss = per_sample[mask].mean()        # padding 제외
```

**Q. loss가 매 step 반드시 내려가야 하나?**

아니다. 미니배치마다 데이터가 다르므로 **출렁이는 게 정상**이다. 판단은 에폭 평균이나 이동평균으로 한다.

**Q. 서로 다른 loss의 숫자를 비교해도 되나?**

안 된다. reduction, 손실 종류, 출력 스케일이 다르면 숫자의 의미가 다르다. MSE 0.3과 CrossEntropy 0.3은 아무 관계가 없다.

**Q. loss와 metric은 왜 다를 수 있나?**

- **loss** — 미분 가능해야 한다. 학습을 굴리는 연료
- **metric** — 미분 불가능해도 된다. 사람이 판단하는 기준

확률이 0.45에서 0.49로 올라도 임계값 0.5를 못 넘으면 accuracy는 그대로다. loss는 내려가는데 metric은 안 움직이는 구간이 생기는 이유다.

> 학습은 **loss**로, 모델 선택은 **metric**으로.

---

## 2. 문제 유형별 손실 함수 선택

**손실 함수 선택은 출력층 설계와 한 세트다.** 마지막 Linear의 `out_features`, target의 dtype·shape, loss가 서로 맞아야 한다.

**순서대로 답할 네 가지 질문**

1. 출력이 **실수**인가 **범주**인가?
2. 범주라면 클래스가 **몇 개**인가?
3. 클래스가 **상호 배타적**인가? (하나만 정답 vs 여러 개 동시 정답)
4. 마지막 층에 활성화를 붙일 것인가? → **대부분 붙이지 않는다**

|문제|out_features|target dtype / shape|loss|
|---|---|---|---|
|회귀|1 (또는 출력 개수)|`float`, `(N, 1)`|`MSELoss`|
|이진 분류|1|`float`, `(N, 1)`|`BCEWithLogitsLoss`|
|다중 분류|C|`long`, `(N,)` — 클래스 인덱스|`CrossEntropyLoss`|
|다중 레이블|C|`float`, `(N, C)` — 0/1|`BCEWithLogitsLoss`|

**Q. 왜 Sigmoid + BCELoss가 아니라 BCEWithLogitsLoss인가?**

수치 안정성. sigmoid 출력이 0이나 1에 가까워지면 `log(0)`이 되어 `inf`나 `nan`이 뜬다. `BCEWithLogitsLoss`는 두 연산을 합쳐 log-sum-exp 방식으로 안전하게 계산한다. **모델은 logit을 그대로 뱉고, loss가 알아서 처리한다.**

**Q. 왜 이진 분류 target이 float인가?**

0/1뿐 아니라 0.3 같은 **soft label**도 받도록 설계되었기 때문이다. logit과 dtype도 맞아야 한다. 반면 `CrossEntropyLoss`의 target은 인덱스이므로 `long`이다.

**Q. CrossEntropyLoss에 softmax를 붙여야 하나?**

**붙이면 안 된다.** 내부에 `log_softmax`가 이미 들어 있다. 두 번 적용하면 분포가 뭉개져 학습이 느려지거나 멈춘다.

**Q. class index를 주소처럼 읽는다는 게 무슨 뜻?**

target이 `3`이면 값 3이 아니라 **"3번 칸"**이라는 위치다. C개 출력 중 3번째의 로그 확률을 가져오라는 지시. one-hot으로 바꿀 필요가 없고, 메모리·속도 면에서 인덱스가 유리하다.

**Q. 클래스가 2개여도 CrossEntropyLoss를 쓸 수 있나?**

쓸 수 있다. 두 설계 모두 맞다.

```python
# 방법 A — out=1, BCEWithLogitsLoss, target float (N,1)
# 방법 B — out=2, CrossEntropyLoss,  target long  (N,)
```

**섞으면 안 된다.** out=1인데 CrossEntropyLoss를 쓰면 클래스가 하나뿐이라 항상 같은 답이 나온다.

**Q. pos_weight는 언제 등장하나?**

이진 분류에서 **양성이 희귀할 때** 양성 샘플의 loss에 가중치를 준다.

```python
pos_weight = torch.tensor([n_negative / n_positive])
crit = nn.BCEWithLogitsLoss(pos_weight=pos_weight)
```

앞 단원의 "놓치면 안 되는 문제에서는 recall 우선"과 같은 이야기다. 임계값 조정이 예측 _후_ 처리라면, `pos_weight`는 학습 _중_에 같은 목적을 달성한다.

---

## 3. Optimizer와 learning rate

**Q. Optimizer는 무엇을 하나?**

역할이 셋으로 나뉜다.

|담당|하는 일|
|---|---|
|loss|얼마나 틀렸는지 숫자 하나로|
|autograd (`backward`)|각 parameter를 조금 늘리면 loss가 어떻게 변하는지|
|optimizer (`step`)|그 정보로 실제 값을 얼마나 바꿀지|

**Q. Learning rate란?**

한 step에 gradient 방향으로 **얼마나 갈지 정하는 배율**

```
새 값 = 현재 값 − lr × (gradient에서 파생된 이동량)
```

**Q. 너무 큰 lr의 overshoot**

최솟값을 지나쳐 반대편으로 튀고, 그게 반복되면 발산한다. loss가 계속 커지거나 `nan`이 된다.

**Q. 너무 작은 lr도 안전하기만 한 건 아니다**

- 학습이 끝나지 않는다 (시간 낭비)
- 얕은 국소 최소점이나 평평한 구간에서 빠져나오지 못한다
- loss curve가 거의 수평이라 "학습이 안 되는 것"처럼 보인다

**Q. 적절한 값은 고정된 공식이 아니다**

모델 구조, 데이터, 배치 크기, optimizer에 따라 다르다. 관례적 출발점은 **Adam 1e-3, SGD 1e-2 ~ 1e-1** 정도.

**Q. lr 비교 실험은 어떻게 하나?**

1. **초기 parameter를 같게 만든다** — seed를 고정하지 않으면 lr 효과인지 초기값 효과인지 구분할 수 없다
2. **first loss가 같은지 먼저 확인한다** — 다르면 조건이 안 맞은 것
3. **curve 모양을 읽는다**

|curve|해석|
|---|---|
|위로 치솟음 / nan|lr 과대, 발산|
|톱니처럼 진동|lr 다소 큼|
|거의 수평|lr 과소|
|빠르게 내려가다 완만|적절|

4. **큰 lr에서 발산했다고 항상 코드 오류는 아니다** — 정상적인 현상이다

**Q. SGD의 stochastic은 무슨 뜻?**

optimizer 이름이라기보다 **gradient를 어떻게 계산하는지**와 연결된 말이다. 전체 데이터가 아니라 미니배치로 gradient를 _추정_하기 때문에 "확률적"이다. 그래서 loss가 출렁이는 것도 자연스럽다.

**Q. momentum을 넣으면 무엇이 달라지나?**

이전 이동 방향을 **관성처럼 누적**한다. 좁은 골짜기에서 좌우 진동을 상쇄하고 바닥 방향으로 빠르게 나아간다. 보통 `momentum=0.9`.

**Q. SGD가 단순하다는 게 약하다는 뜻은 아니다**

잘 튜닝된 SGD + momentum이 Adam보다 **일반화 성능이 좋은 경우가 흔하다.** 특히 이미지 분야에서 그렇다.

**Q. Adam은 무엇을 기억하나?**

parameter마다 두 개의 상태를 저장한다.

- **1차 모멘트** — gradient의 이동평균 (방향)
- **2차 모멘트** — gradient 제곱의 이동평균 (크기)

2차 모멘트로 나누기 때문에 **자주 크게 흔들린 parameter는 조심히, 거의 안 움직인 parameter는 과감히** 움직인다. 그래서 같은 lr이어도 parameter마다 실제 step이 달라진다.

```python
model = Net()
optimizer = optim.Adam(model.parameters(), lr=1e-3)   # 학습 loop 밖에서 한 번만
```

**Q. parameter group은 언제 쓰나?**

층마다 다른 lr을 주고 싶을 때.

```python
optimizer = optim.Adam([
    {'params': model.backbone.parameters(), 'lr': 1e-4},
    {'params': model.head.parameters(),     'lr': 1e-3},
])
```

**Q. checkpoint에 optimizer state가 왜 필요한가?**

이어서 학습할 때 momentum·모멘트를 복원해야 하기 때문이다. 없으면 재시작 직후 loss가 튄다.

```python
torch.save({
    'model': model.state_dict(),
    'optimizer': optimizer.state_dict(),
    'epoch': epoch,
}, path)
```

---

## 4. 학습 step 순서

```python
for x, y in loader:
    optimizer.zero_grad()      # 1. 이전 gradient 비우기
    pred = model(x)            # 2. forward — prediction + 계산 그래프
    loss = criterion(pred, y)  # 3. loss — 스칼라 하나
    loss.backward()            # 4. .grad 채우기
    optimizer.step()           # 5. parameter 값 변경
```

**각 단계에서 무엇이 생기나**

|단계|생기는 것|
|---|---|
|forward|prediction, 계산 그래프|
|loss|스칼라 텐서 (`grad_fn` 보유)|
|backward|각 parameter의 `.grad`|
|step|parameter 값 자체가 바뀜|

**Q. loss는 step 후 자동으로 다시 계산되나?**

아니다. `loss`는 **그 시점의 스냅샷**이다. step 이후의 loss를 보려면 다시 forward를 해야 한다.

**Q. zero_grad를 안 하면?**

gradient가 **덧셈으로 누적**된다. 두 번째 step에서는 첫 번째 gradient까지 섞인 방향으로 이동한다.

```python
loss.backward(); print(w.grad)   # 예: 2.0
loss.backward(); print(w.grad)   # 4.0 — 초기화 안 하면 누적
```

**Q. gradient accumulation은 언제 의도적으로 쓰나?**

메모리가 부족해 큰 배치를 못 쓸 때, 여러 미니배치의 gradient를 모아 큰 배치처럼 만든다.

```python
for i, (x, y) in enumerate(loader):
    loss = criterion(model(x), y) / accum_steps   # loss를 나눠야 스케일이 맞는다
    loss.backward()
    if (i + 1) % accum_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

**Q. None과 0 Tensor의 차이는?**

`zero_grad()`의 기본값은 `set_to_none=True`라서 `.grad`가 **None이 된다.** 0으로 채운 텐서보다 메모리를 아끼고 조금 빠르다. 다만 `.grad`가 None인 parameter는 optimizer가 건너뛰므로, `None`인지 `0`인지에 따라 동작이 미묘하게 다르다.

**Q. loss.item()에는 왜 backward할 수 없나?**

`.item()`은 Python `float`을 반환한다. 계산 그래프와 완전히 끊어져 있으므로 미분할 대상이 없다.

**4단계 디버깅 패턴**

|단계|확인할 것|
|---|---|
|1|입력·출력 계약 — shape, dtype이 loss 요구와 맞는가|
|2|loss가 유한한가 (`nan`, `inf` 아닌가)|
|3|gradient가 존재하고 유한한가 (`None` 아닌가)|
|4|parameter가 실제로 바뀌는가|

위에서부터 순서대로 본다. 3번이 `None`이면 그래프가 끊긴 것이고, 4번만 안 되면 optimizer 설정 문제다.

---
## 한 줄 정리

> **loss**는 얼마나 틀렸는지, **gradient**는 어느 방향인지, **optimizer**는 얼마나 갈지를 담당한다. 세 역할이 분리되어 있으므로, 학습이 안 될 때도 **어느 단계에서 끊겼는지** 순서대로 짚으면 된다.

**다음에 볼 것** — learning rate scheduler, AdamW와 weight decay, label smoothing, gradient clipping

---

## 참고 링크

**PyTorch 공식 문서**

- [torch.nn.MSELoss](https://docs.pytorch.org/docs/stable/generated/torch.nn.MSELoss.html)
- [torch.nn.BCEWithLogitsLoss](https://docs.pytorch.org/docs/stable/generated/torch.nn.BCEWithLogitsLoss.html) — `pos_weight`, 수치 안정성 설명
- [torch.nn.BCELoss](https://docs.pytorch.org/docs/stable/generated/torch.nn.BCELoss.html) — 비교용
- [torch.nn.CrossEntropyLoss](https://docs.pytorch.org/docs/stable/generated/torch.nn.CrossEntropyLoss.html) — target shape·dtype, `label_smoothing`
- [torch.optim](https://docs.pytorch.org/docs/stable/optim.html) — optimizer 사용법, parameter group, scheduler
- [torch.optim.SGD](https://docs.pytorch.org/docs/stable/generated/torch.optim.SGD.html) — momentum 수식
- [torch.optim.Adam](https://docs.pytorch.org/docs/stable/generated/torch.optim.Adam.html)
- [Autograd mechanics](https://docs.pytorch.org/docs/stable/notes/autograd.html) — 그래프 생성·해제, `detach`

**공식 튜토리얼**

- [Optimizing Model Parameters](https://docs.pytorch.org/tutorials/beginner/basics/optimization_tutorial.html) — `zero_grad → backward → step` 3단계 설명

**논문**

- Kingma & Ba (2015). [Adam: A Method for Stochastic Optimization](https://arxiv.org/abs/1412.6980) — ICLR
- Loshchilov & Hutter (2019). [Decoupled Weight Decay Regularization](https://arxiv.org/abs/1711.05101) — AdamW