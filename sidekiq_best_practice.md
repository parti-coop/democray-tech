# Sidekiq Best Practices

Sidekiq은 redis를 사용하여 작업 큐, 작업 상태, 작업 결과등의 정보를 저장하고 관리, 작업을 스케줄링하고, 작업 상태를 추적하고, 작업 결과를 저장하고 조회하는 기능을 가집니다.
Sidekiq 위키에 [Best Practices](https://github.com/sidekiq/sidekiq/wiki/Best-Practices) 라는 페이지가 있어서 소개해보고자 합니다. 총 4개의 섹션으로 나뉘어져 있고, 각각 sidekiq을 어떻게 사용해야 하는 지, 또는 어떤 개념을 가지고 있어야 하는 지를 설명하고 있습니다.

> 1. Make your job parameters small and simple

sidekiq의 파라메터를 작게 가져가야 한다고 합니다. perform_sync를 통해 sidekiq의 job을 실행합니다.

```ruby
SomeJob.perform_sync(quote_id)
```

라고 실행하여 Job을 실행하는데, 그 파라메터 datatype은 string, integer, float, boolean, nil, array, hash가 가능합니다. 그러나 ruby symbols은 넘겨주지 못합니다. Sidekiq client API는 JSON.dump를 사용해 Redis서버로 데이타를 넘기고, Sidekiq server API는 JSON.load로 변수를 변환합니다. 이 와중에 symbols, named parameters, keyword arguments는 이 dump/load 과정에서 제대로 넘겨지지 못합니다. ruby symbol을 넘겨주려면 [Hash#stringify_keys](https://apidock.com/rails/Hash/stringify_keys)를 사용하면 됩니다.

아무래도 job에서 파라메터는 성능과 직결한 것이니, 변환에 조금이라도 시간이 걸리는 것을 제거하는 것이 성능 측정해서도 좋겠지만, 메모리 사용량이나, 유지보수, 확장성 측면에서도 파라메터의 값이 단순해야 할 것입니다.

<details>
<summary>리팩토링 할만한건?</summary>
기존에 파라메터가 좀 복잡한 job이 있다면파라메터를 id하나로만 넘기고 job을 여러개로 만드는 방식으로 refactoring하는 방안을 생각해보면 될것같습니다.
</details>


> 2. Make your job idempotent and transactional

이 테크노트를 쓰게 된 이유가 여기에 있습니다. idempotent가 그 이유가 되는 용어인데요. 이 뜻은 여러번 실행을 해도 똑같은 결과값이 나온다는 뜻입니다. idempotency라는 말은 수학용어에서는 멱함수라고 합니다. 어떤 연산이나 함수가 동일한 입력에 대해 여러 번 적용하더라도 결과가 변하지 않는 성질을 말합니다. 이를 job 프로세스에 적용하면, 동일한 작업이 여러 번 실행 되더라도 동일 한 결과가 나와야 하며, 시스템의 상태가 변하지 않아야 합니다.

이를 통해서 재시도 관리, 중복 처리 관리, 일관성 보장, 시스템 안정성 향상등의 이점을 얻을 수 있습니다.

```ruby
def perform(card_charge_id)
  charge = CardCharge.find(card_charge_id)
  charge.void_transaction
  Emailer.charge_refunded(charge).deliver
end
```

위의 예시처럼 만약 credit 카드가 환불되었고 그걸 메일로 보낼 때, void_transaction을 통해 이미 환불되었다면 이를 적절하게 처리할 수 있도록 해야 합니다.

<details>
<summary>리팩토링 할만한건?</summary>

1. 상태 관리: 중복작업/재시도시에도 일관성이 유지되었는지 확인가능해야 합니다.
2. 작업이 중간에 실패하거나 중단되도 시스템이 일관성을 유지되도록 해야 합니다.
3. 작업 실패한 후에도 작업을 재시도하는 로직을 추가할 수도 있어야 합니다.

</details>

> 3. Embrace Concurrency

Sidekiq은 병렬 실행을 위해 설계 되었습니다. 작업이 병렬로 실행 할 수 있도록 작업 설계하길 바랍니다.

예를 들어 Sidekiq에는 [Concurrency를 조정](https://github.com/sidekiq/sidekiq/wiki/Advanced-Options#concurrency)할 수 있습니다. 기본으로는 10개의 thread를 가지고 있지만 이게 버든이 된다면, 낮게 조정할 수 도 있습니다. 기본적으로 50이상으로 세팅하는 것은 권장하지 않습니다.


> 4. Use Precise Terminology
