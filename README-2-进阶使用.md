[前文](README.md)介绍了 DP 的第一个原则：`将隐性的概念显性化`。在这里我将用一个新的案例介绍 DP 的另外两个原则。

# Domain Primitive 三条原则如下

- 将隐性的概念显性化（Make Implicit Concepts Explicit）
- 让隐性的上下文显性化（Make Implicit Context Explicit）
- 封装多对象行为（Encapsulate Multi-Object Behavior）

## 案例1 - 汇款
假设现在要实现一个汇款功能，让A用户可以支付 x 元给用户 B ，可能的实现如下：
```
public void pay(BigDecimal money, Long recipientId) {
    BankService.transfer(money, "CNY", recipientId);
}
```
如果这个是境内转账，并且境内的货币永远不变，该方法貌似没啥问题，
但如果有一天货币变更了（比如欧元区曾经出现的问题），或者我们需要做跨境转账时，该方法才会暴露出功能缺陷。
因为 money 对应的货币不一定是人民币"CNY"，更可能是外币。

在这个 case 里，当我们说`支付 x 元`时，除了 x 本身的数字之外，实际上还有一个隐含的概念，那就是货币币种为人民币。
但是在原始的入参里，之所以只用了 BigDecimal 的原因是我们认为 CNY 货币是默认的，是一个隐含的条件。
但是在我们写代码时，需要把所有隐性的条件显性化，而这些条件整体组成当前的上下文。

所以 DP 的第二个原则是：

> 将隐性的上下文显性化（Make Implicit Context Explicit）

所以当我们做这个支付功能时，实际上需要的一个入参是支付金额 + 币种。

我们可以把这两个概念组合成为一个独立的完整概念：Money。

```
@Value
public class Money {
    private BigDecimal amount;
    private Currency currency;
    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
}

// 而原有的代码则变为：
public void pay(Money money, Long recipientId) {
    BankService.transfer(money, recipientId);
}
```

## 案例2 - 跨境汇款

前面的案例升级一下，假设用户可能要做跨境转账，从人民币CNY到美元USD，并且货币汇率随时在波动：

```
public void pay(Money money, Currency targetCurrency, Long recipientId) {
    if (money.getCurrency().equals(targetCurrency)) {
        BankService.transfer(money, recipientId);
    } else {
        BigDecimal rate = ExchangeService.getRate(money.getCurrency(), targetCurrency);
        BigDecimal targetAmount = money.getAmount().multiply(new BigDecimal(rate));
        Money targetMoney = new Money(targetAmount, targetCurrency);
        BankService.transfer(targetMoney, recipientId);
    }
}
```
在跨境汇款转账的例子里，由于目标币种`targetCurrency`不一定和原币种一致，所以需要调用一个服务去查询实时汇率，然后做计算。最后用计算后的结果做汇款转账。

这里最大的问题在于，金额的计算被包含在了支付的服务中，涉及到的对象也有2个 Currency ，2 个 Money ，1 个 BigDecimal ，总共 5 个对象。这种涉及到多个对象的业务逻辑，需要用 DP 包装掉，所以这里引出 DP 的第三个原则：

> 封装多对象行为（Encapsulate Multi-Object Behavior）

在这个案例里，我们可以将转换汇率的功能封装到一个叫做 ExchangeRate 的 DP 里：
```java
@Value
public class ExchangeRate {
    private BigDecimal rate;
    private Currency from;
    private Currency to;

    public ExchangeRate(BigDecimal rate, Currency from, Currency to) {
        this.rate = rate;
        this.from = from;
        this.to = to;
    }

    public Money exchange(Money fromMoney) {
        notNull(fromMoney);
        isTrue(this.from.equals(fromMoney.getCurrency()));
        BigDecimal targetAmount = fromMoney.getAmount().multiply(rate);
        return new Money(targetAmount, to);
    }
}
```

使用汇率对象`ExchangeRate`，通过封装金额计算逻辑以及各种校验逻辑，让原始代码变得极其简单：

```
public void pay(Money money, Currency targetCurrency, Long recipientId) {
    ExchangeRate rate = ExchangeService.getRate(money.getCurrency(), targetCurrency);
    Money targetMoney = rate.exchange(money);
    BankService.transfer(targetMoney, recipientId);
}
```
