
# 왜 비동기인가?
- 메인 스레드가 멈추지 않게 하여, 메인 스레드가 방해 받지 않게 함으로서 시간적 효율을 늘리는 목적으로 비동기식 프로그래밍을 사용합니다.
- `JavaScript`의 경우, `Promise/Async-Await`기반의 비동기식 통제를 사용한다고 생각하시면 되고, C#의 경우 좀 더 고급화된 기법의 비동기식 프로그래밍을 지원하고 있습니다.

# Task/Func 객체

### Task? Func?
- 보통 비동기형 함수를 다루는 `Javascript`의 `Promise`와 상동한 것은 같습니다.
- Task는 return값이 `void` 인 경우, Func는 return 값이 `void`가 아닌 경우 사용하는 것으로 이해하면 좋습니다.
## Await-Async 

### Await-Async와 Task를 사용할 때 주의점

![[../image/Pasted image 20240212142835.png]]

- `.ConfigureAwait(false);`를 통해서 `await` 구문을  `task`형식으로 바꾸어주면 해결이 되긴 합니다.
- 이 함수는 `await/async`가 사용하는 `SynchronizationContext` Class에서 Task가 종료되길 기다리지 않고 바로 실행시켜주는 
> `SynchronizationContext` : 이 class는 async기반 통제에서 비동기식 작업의 queue를 가지고 있습니다. 
> async-await 기반은 스레드가 점유되어 있다고 판단하면 그 잡겁은 시행하지 않습니다.
> 이는, `async-await` 기반 함수가 점유권 문제로 부모 Task작업이 종료되지 않는다면 자기가 절대 실행되지 않는 교착상태에 빠짐을 의미합니다.
### 번외) UniTask
- Unity에서 Task를 Await/Async기반으로 쓸 수 있게 해주는 라이브러리입니다.

# 멀티스레딩, `Thread`

## 멀티스레드의 통제기법


### 임계구역(Critical Zone)의 문제와 그 해결

#### 임계구역(Critical Zone) -> Race Condition
- 이하의 코드를 동작시키면 **엉뚱한 값**이 나옵니다.
```C#
using System;
using System.Threading;

class Program
{
    static int sharedResource = 0;

    static void Main(string[] args)
    {
        // 스레드 A
        new Thread(() =>
        {
            for (int i = 0; i < 5; i++)
            {
                // 공유 자원에 대한 접근
                sharedResource++;
                Console.WriteLine($"스레드 A: 공유 자원 증가 - {sharedResource}");
            }
        }).Start();

        // 스레드 B
        new Thread(() =>
        {
            for (int i = 0; i < 5; i++)
            {
                // 공유 자원에 대한 접근
                sharedResource--;
                Console.WriteLine($"스레드 B: 공유 자원 감소 - {sharedResource}");
            }
        }).Start();
    }
}

```
![[../image/Pasted image 20240212151653.png]]
- 그 이유는 **하나의 메모리에 여러 스레드가 개입했기 때문입니다.**
- 

#### Locking...
- 이 때문에, Lock을 통해서 

- 간단히 말하자면, 메모리가 점유되어 있다는 표식을 하여 
```C#
lock(object) // 통제를 위한 세마포어
{
// 통제환경에서 실행되는 함수

}
```
- 이 함수는 `object`가 **잠김상태가 아닐때**에만 lock문 아래로 진입하는 조건을 가집니다.
- 또한, lock문에 진입하면 
이때, lock문이 발동되면 `object`는 자동으로 잠김상태가 된다는 점을 알아둡시다.
#### Mutex

#### Semaphore

#### Semaphore In C\#, `Monitor`

##### `Wait(obj)`

##### `Pulse(obj)`

##### `Enter(obj)`, `Exit(obj)`
- 개인적으로는 lock을 쓰는게 더 깔끔하고 나아보인다만, 필요하다면 사용할 수도 있다로 알아두는게 Thread-Safe하다고 생각됩니다.

## `Lazy` Class
- **싱글턴 패턴**은 하나의 메모리를 기반으로 움직이는 것이라서, 메모리의 동시접근 문제가 생길 수 있습니다.
- 이를 

## C\#스레딩 기법
```c#
using System;
using System.Threading.Tasks;

public class Account
{
    private readonly object balanceLock = new object();
    private decimal balance;

    public Account(decimal initialBalance) => balance = initialBalance;

    public decimal Debit(decimal amount)
    {
        if (amount < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(amount), "The debit amount cannot be negative.");
        }

        decimal appliedAmount = 0;
        lock (balanceLock)
        {
            if (balance >= amount)
            {
                balance -= amount;
                appliedAmount = amount;
            }
        }
        return appliedAmount;
    }

    public void Credit(decimal amount)
    {
        if (amount < 0)
        {
            throw new ArgumentOutOfRangeException(nameof(amount), "The credit amount cannot be negative.");
        }

        lock (balanceLock)
        {
            balance += amount;
        }
    }

    public decimal GetBalance()
    {
        lock (balanceLock)
        {
            return balance;
        }
    }
}

class AccountTest
{
    static async Task Main()
    {
        var account = new Account(1000);
        var tasks = new Task[100];
        for (int i = 0; i < tasks.Length; i++)
        {
            tasks[i] = Task.Run(() => Update(account));
        }
        await Task.WhenAll(tasks);
        Console.WriteLine($"Account's balance is {account.GetBalance()}");
        // Output:
        // Account's balance is 2000
    }

    static void Update(Account account)
    {
        decimal[] amounts = [0, 2, -3, 6, -2, -1, 8, -5, 11, -6];
        foreach (var amount in amounts)
        {
            if (amount >= 0)
            {
                account.Credit(amount);
            }
            else
            {
                account.Debit(Math.Abs(amount));
            }
        }
    }
}
```