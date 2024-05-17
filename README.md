参考: https://gitee.com/robbinqin/playground2022
- 原文：《阿里技术专家详解 DDD 系列 第一讲- Domain Primitive》
- 作者：殷浩
- 发布日期：2019年9月2日

# 总目录
阿里技术专家殷浩详解DDD系列
- DDD系列 第一讲：Domain Primitive
  - https://juejin.cn/post/6844904177207001101
- DDD系列 第二讲：应用架构
  - https://juejin.cn/post/6844904201575743495
- DDD系列 第三讲：Repository模式
  - https://juejin.cn/post/6845166890554228744
- DDD系列 第四讲：领域层设计规范
  - https://juejin.cn/post/6912228908075057166
- DDD系列 第五讲：聊聊如何避免写流水账代码
  - https://developer.aliyun.com/article/783664

---

# Domain Primitive
今天先给大家带来一篇最基础，但极其有价值的Domain Primitive的概念。

就好像在学任何语言时首先需要了解的是基础数据类型一样，在全面了解 DDD 之前，首先给大家介绍一个最基础的概念: Domain Primitive（以下将Domain Primitive缩写为DP）。

> Primitive 这个词的直译是“原语”

就好像 Integer、String 是所有编程语言的Primitive一样，在 DDD 里， DP 可以说是一切模型、方法、架构的基础，而就像 Integer、String 一样， DP 又是无所不在的。所以，第一讲会对 DP 做一个全面的介绍和分析，但我们先不去讲概念，而是从案例入手，看看为什么 DP 是一个强大的概念。

# 引入问题

我们先举例子，某某客户注册系统，业务逻辑如下：

> 某公司通过各省市的`地推业务员`邀请企业客户注册的模式进行商业推广，需要做一个客户注册系统。 要求按新客注册时登记的企业电话号码的区号进行客源区域划分，核算所属区域对应的地推业务团队的工作业绩。（假定要求地推业务员必须登记客户的企业座机号码）

先不要去纠结这个根据客户电话去发奖金的业务逻辑是否合理，也先不要去管客户是否应该在注册时和业务员做绑定，这里我们看的主要还是如何更加合理的去实现这个逻辑。一个简单的新客注册和配置销售代表的代码实现如下：

### RegistrationService.java

```java
public interface RegistrationService {
    ResultDto register(String name, String telephone, String address);
}
```

### RegistrationServiceImpl.java

```java
public class RegistrationServiceImpl implements RegistrationService {

    private SalesManagerRepository managerRepo;
    private CustomerRepository customerRepo;

    public CustomerDTO register(String name, String telephone, String address) throws ValidationException {
        // 入参校验
        if (name == null || name.length() == 0) {
            throw new ValidationException("name");
        }
        if (telephone == null || !isValidTelephoneNumber(telephone)) {
            throw new ValidationException("telephone");
        }

        // 取电话号里的长途区号，然后通过区号找到区域内的地推销售代表
        String areaCode = null;
        Set<String> areas = new HashSet<>(Arrays.asList("0571", "021", "010"));
        for (int i = 0; i < telephone.length(); i++) {
            String prefix = telephone.substring(0, i);
            if (areas.contains(prefix)) {
                areaCode = prefix;
                break;
            }
        }
        SalesManager relatedSalesman = managerRepo.findAreaSalesManager(areaCode);

        // 创建1条客户信息，落库，然后返回
        Customer c = new Customer();
        c.name = name;
        c.telephone = telephone;
        c.address = address;
        c.memberSince = LocalDate.now();
        c.relatedSalesManagerId = relatedSalesman.getId();
        Customer savedCustomer = customerRepo.save(c);
        return CustomerDTO.of(savedCustomer).with(relatedSalesman);
    }

    private static boolean isValidTelephoneNumber(String telephone) {
        String pattern = "^0[1-9]\\d{1,2}-?\\d{8}$";
        return telephone.matches(pattern);
    }
}
```

### Customer.java

```java
class Customer {
    Long id; // 客户ID,作为数据库主键
    String name; // 客户姓名或公司名
    String telephone; // 客户公司座机号码(必填项)
    String address; // 客户联系地址(可选)
    LocalDate memberSince; // 注册成功起始日期YYYY-MM-DD

    Long relatedSalesManagerId; // 对应的专属销售代表的ID

    // 省略get/set方法等其他内容
}
```

### SalesManager.java

```java
class SalesManager {
    Long id; // 地推销售代表的ID
    String name; // 销售代表姓名
    String area; // 负责的区域
    // 省略其他内容...
}
```

# 业务代码问题分析

我们日常绝大部分代码和模型其实都与如上实例相似乍一看貌似没啥问题。但我们再深入一步，从以下四个维度去分析一下：

- 接口的清晰度（可阅读性）
- 数据验证和错误处理
- 业务逻辑代码的清晰度
- 可测试性

## 问题1 - 接口的清晰度（可阅读性）
在Java代码中，对于一个方法来说所有的参数名在编译时丢失，留下的仅仅是一个参数类型的列表，所以我们重新看一下以上的接口定义，其实在运行时仅仅是：

```java
Customer register(String name, String telephone, String address);
```

所以如下一行代码：

```
service.register("殷浩", "浙江省杭州市余杭区文三西路969号", "0571-12345678");
// 备注：人眼阅读代码时很难发现此类bug，
// Java编译器也无法检测同为String类型的3个入参的先后顺序是否颠倒。
```

当然，在真实代码中运行时会报错，但这种 bug 是在运行时被发现的，而不是在编译时。普通的 Code Review 也很难发现这种问题。往往要等到代码上线运行后才会被暴露出来。

有没有办法在编码时就避免这种可能会出现的问题呢？

另外一种常见的，特别是在查询服务中容易出现的例子如下：

```java
public interface CustomerRepository {
    Customer findByName(String name);
    Customer findByTelephone(String telephone);
    Customer findByNameAndTelephone(String name, String telephone);
}
```

在这个场景下，由于入参都是 String 类型，不得不在方法名上面加上 `By某某某` 来区分，而 `findByNameAndTelephone(String,String)` 同样也会陷入前面的入参顺序错误的问题，而且和前面的入参不同，这里参数顺序如果输错了，方法不会报错只会返回 null，而这种 bug 更加难被发现。

有没有办法让方法入参一目了然，避免入参错误导致的bug？

## 问题2 - 数据验证和错误处理

在前面这段数据校验代码：

```
if (telephone == null || !isValidTelephoneNumber(telephone)) {
    throw new ValidationException("telephone");
}
```

在日常编码中经常会出现，一般来说这种代码需要出现在方法的最前端，确保能够 fail-fast 。但是假设你有多个类似的接口和类似的入参，在每个方法里会重复出现这种入参校验代码。
而更严重的是，如果未来我们要拓展电话号码去包含手机时，很可能需要加入以下代码：

```
if ( phone == null
    || !isValidTelephoneNumber(phone)
    || !isValidMobilePhoneNumber(phone)) {
    throw new ValidationException("phone");
}
```

如果你有很多个地方用到了电话号码这个入参，改好其中一个地方，但是还有其他地方漏了没有同时修改，就会造成bug。

出现这种现象说明之前没有合并同类项，优化代码复用，违反了DRY原则(Don't Repeat Yourself)。

与其他模块对接联调时，如果提出新的需求（例如要求我们把入参错误的详细原因一并返回给前端），那么这段代码就变得更加复杂：

```
if (telephone == null) {
    throw new ValidationException("telephone: 电话号码不能为空");
} else if (!isValidTelephoneNumber(telephone)) {
    throw new ValidationException("telephone: 电话号码格式错误");
}
```

可以想像得到，代码里充斥着大量的类似代码块时，维护成本要有多高。

最后，这个业务方法里会抛出`ValidationException`，所以需要外部调用方去try/catch。而业务逻辑异常和数据校验异常被混在了一起，是否合理呢？

在传统Java架构里有几个传统的方法能够缓解一部分问题，常见的如`@NotNull @NotBlank @Pattern(regexp)`等注解或自己写`ValidationUtils`工具类，比如：

```
// 使用 @NotNull 注解
public Customer register(
    @NotNull @NotBlank                                     String name,
    @NotNull @Pattern(regexp = "^0[1-9]\\d{1,2}-?\\d{8}$") String telephone,
    @NotNull                                               String address) { /* */ }
```
```
// 使用 ValidationUtils 类:
public Costumer register(String name, String telephone, String address) {
    ValidationUtils.validateName(name); // throws ValidationException
    ValidationUtils.validateTelephone(telephone);
    ValidationUtils.validateAddress(address);
}
```

但这几个传统的方法同样有问题：

- 使用`@NotNull @NotBlank`注解：
通常只能解决简单的校验逻辑，复杂的校验逻辑一样要写代码实现定制校验器。
在添加了新校验逻辑时，同样会出现在某些地方忘记添加一个注解的情况，DRY原则还是会被违背。

- 自己写`ValidationUtils`工具类：
当大量的校验逻辑集中在一个类里之后，违背了单一责任原则，导致代码混乱和不可维护。
业务异常和校验异常还是会混杂。

那么到底有没有能够一劳永逸的解决所有校验的问题的方法？同时还能降低后续的维护成本和异常处理成本呢？

## 问题3 - 业务代码的清晰度

分析如下代码段：
```
String areaCode = null;
Set<String> areas = new HashSet<>(Arrays.asList("0571", "021", "010"));
for (int i = 0; i < telephone.length(); i++) {
    String prefix = telephone.substring(0, i);
    if (areas.contains(prefix)) {
        areaCode = prefix;
        break;
    }
}
SalesRep rep = salesRepRepo.findRep(areaCode);
```
实际项目中还会另外一种常见的情况，那就是从一些入参里抽取一部分数据，然后调用一个外部依赖获取更多的数据，然后通常从新的数据中再抽取部分数据用作其他的作用。
这种代码通常被称作“胶水代码”，其本质是由于外部依赖的服务的入参并不符合我们原始的入参导致的。
比如，如果SalesRepRepository包含一个findRepByPhone的方法，则上面大部分的代码都不必要了。

所以，一个常见的办法是将这段代码抽离出来，变成独立的一个或多个方法：

```
private static String findAreaCode(String telephone) {
    for (int i = 0; i < telephone.length(); i++) {
        String prefix = telephone.substring(0, i);
        if (isAreaCode(prefix)) {
            return prefix;
        }
    }
    return null;
}

private static boolean isAreaCode(String prefix) {
    final Set<String> areas = new HashSet<>(Arrays.asList("0571", "021", "010"));
    return areas.contains(prefix);
}
```

然后原始代码变为：

```
String areaCode = findAreaCode(telephone);
SalesRep rep = salesRepRepo.findRep(areaCode);
```

而为了复用以上的方法，可能会抽离出一个静态工具类 PhoneUtils 。

但是这里仍要思考：编写静态工具类的方案是不是最好的实现方式呢？

当你的项目里充斥着大量的静态工具类，业务代码散在多个文件当中时，你是否还能找到核心的业务逻辑呢？

## 问题4 - 可测试性
为了保证代码质量，每个方法里的每个入参的每个可能出现的条件都要有测试用例覆盖（假设我们先不去测试内部业务逻辑）。
在我们这个方法里需要以下的TC测试用例表：

![](./assets/图1.png)

假如一个方法有 N 个参数，每个参数有 M 个校验逻辑，至少要有 N * M 条单元测试。

如果这时候在该方法中加入一个新的入参字段 fax 表示传真机电话号码，即使 fax 和 telephone 的校验逻辑完全一致，为了保证测试覆盖率，也一样需要 M 个新的测试用例。

而假设有 P 个方法中都用到了 telephone 这个字段，这 P 个方法都需要对该字段进行测试，也就是说整体需要：

总数 =`P × N × M`个测试用例才能完全覆盖所有数据验证的问题。

在日常项目中，这个测试的成本非常之高，导致大量的代码没被覆盖到。而没被测试覆盖到的代码才是最有可能出现问题的地方。

在这个情况下，降低测试成本 == 提升代码质量，如何能够降低测试的成本呢？

# 解决方案

我们回头先重新看一下原始的 use case，并且标注其中可能重要的概念：

一个新应用在全国通过`地推业务员`做推广，需要做一个客户的注册系统，在客户注册后能够通过长途电话区号核算团队业绩。

在分析了 use case 后，发现其中地推业务员、客户本身自带 ID 属性，属于 Entity（实体），而注册系统属于 Application Service（应用服务），这几个概念已经有存在。
但是发现电话号这个概念却完全被隐藏到了代码之中。

我们可以问一下自己，取电话号的区号的逻辑：是否属于客户？是否属于注册服务？

如果都不是很贴切，那就说明这个逻辑应该属于一个独立的概念。所以这里引入我们第一个原则：

> 将隐性的概念显性化（Make Implicit Concepts Explicit）

在这里，我们可以看到，原来电话号仅仅是用户的一个参数，属于隐形概念，但实际上区号前缀才是真正的业务逻辑。
我们应该将电话号的概念显性化，创建`Value Object`对象：

```java
public class TelephoneNumber {
    private final String number;
    public String getNumber() {
        return number;
    }

    public TelephoneNumber(String number) throws ValidationException {
        if (null == number) {
            throw new ValidationException("number不能为null");
        } else if (!isValid(number)) {
            throw new ValidationException("number格式错误");
        }
        this.number = number;
    }

    public String getAreaCode() {
        for (int i = 0; i < number.length(); i++) {
            String prefix = number.substring(0, i);
            if (isAreaCode(prefix)) {
                return prefix;
            }
        }
        return null;
    }

    private static boolean isAreaCode(String prefix) {
        final Set<String> areas = new HashSet<>(Arrays.asList("0571", "021", "010"));
        return areas.contains(prefix);
    }

    public static boolean isValid(String number) {
        String pattern = "^0?[1-9]{2,3}-?\\d{8}$";
        return number.matches(pattern);
    }
}
```
这里面有几个很重要的元素：

通过`private final`限定`String number`可以确保`PhoneNumber`成为不可改对象（Immutable Value Object）。（一般来说 VO 都是不可改的，这里只是重点强调一下）
校验逻辑都放在了构造器里面，确保只要 PhoneNumber 类对象只要被创建出来就一定是校验通过的。
之前的 `findAreaCode()` 方法变成了 PhoneNumber 类里的 `getAreaCode()` ，突出了 areaCode 是 PhoneNumber 的一个计算属性。
这样做完之后，我们发现把 PhoneNumber 显性化之后，其实是生成了一个`数据类型（Type）`和一个`类（Class）`：

- Type 指我们在今后的代码里可以通过 PhoneNumber 去显性的标识电话号这个概念。
- Class 指我们可以把所有跟电话号相关的逻辑的集中存放到一个文件里。

__这两个概念加起来，构造成了本文标题的 Domain Primitive（DP）。__

我们看一下全面使用了 DP 之后效果：
```java
public class Customer {
    CustomerId customerId;
    Name name;
    PhoneNumber telephone;
    Address address;
    SalesManagerId privateSalesManagerId;
}
```
```java
public class RegistrationServiceImpl implements RegistrationService {

    private SalesManagerRepository salesManagerRepo;
    private CustomerRepository customerRepo;

    public Customer register(
        @NotNull Name name,
        @NotNull PhoneNumber telephone,
        @NotNull Address address
    ) {
        // 找到区域内的SalesManager
        SalesManager relatedSalesman = salesManagerRepo.findSalesmanager(telephone.getAreaCode());

        // 最后创建客户，落盘，然后返回，这部分代码实际上也能用Builder解决
        Customer c = new User();
        c.name = name;
        c.telephone = telephone;
        c.address = address;
        c.relatedSalesManagerId = relatedSalesman.getId();

        return customerRepo.saveCustomer(customer);
    }
}
```
我们可以看到在使用了 DP 之后，所有的数据验证逻辑和非业务流程的逻辑都消失了，剩下都是核心业务逻辑，可以一目了然。我们重新用上面的四个维度评估一下：

### 评估1 - 接口的清晰度
重构后的方法签名变成了很清晰的：

`public Customer register(Name name, PhoneNumber telephone, Address address);`

而之前容易出现的bug，如果按照现在的写法：
```
service.register(new Name("殷浩"), new PhoneNumber("0571-12345678"), new Address("浙江省杭州市余杭区文三西路969号"));
```
让接口 API 变得很干净，易拓展。

### 评估2 - 数据验证和错误处理
```
public Customer register(
    @NotNull Name name,
    @NotNull PhoneNumber telephone,
    @NotNull Address address
) // no throws
```
如前文代码展示的，重构后的方法里，完全没有了任何数据验证的逻辑，也不会抛 `ValidationException` 。

因为 DP 的特性，只要是能够带到入参里的一定是正确的或 null。（另外，Bean Validation 或 lombok 的注解可以解决剩下 null 的问题）。

所以我们把数据验证的工作量前置到了调用方，而调用方本来就是应该提供合法数据的，所以更加合适。

再展开来看，使用DP的另一个好处就是代码遵循了 DRY 原则和单一性原则，如果未来需要修改 PhoneNumber 的校验逻辑，只需要在一个文件里修改即可，所有使用到了 PhoneNumber 的地方都会生效。

### 评估3 - 业务代码的清晰度
```
SalesRep rep = salesRepRepo.findRep(telephone.getAreaCode());
User user = xxx;
return userRepo.save(user);
```
除了在业务方法里不需要校验数据之外，原来的一段胶水代码 findAreaCode 被改为了 PhoneNumber 类的一个计算属性 getAreaCode ，让代码清晰度大大提升。
胶水代码通常都不可复用，但是使用了 DP 后，变成了可复用、可测试的代码。

我们能看到，在刨除了数据验证代码、胶水代码之后，剩下的都是核心业务逻辑。（ Entity 相关的重构在后面文章会谈到，这次先忽略）

### 评估4 - 可测试性

当我们将 PhoneNumber 抽取出来之后，在来看测试的TC测试用例表：
![](./assets/图2.png)

首先 PhoneNumber 本身还是需要 M 个测试用例，但是由于我们只需要测试单一对象，每个用例的代码量会大大降低，维护成本降低。
每个方法里的每个参数，现在只需要覆盖为 null 的情况就可以了，其他的 case 不可能发生（因为只要不是 null 就一定是合法的）。
所以，单个方法的 TC 从原来的 N * M 变成了今天的 N + M 。同样的，多个方法的单元测试总个数变成了 N + M + P。

这个数量一般来说要远低于原来的数量 N × M × P ，让测试成本极大的降低。

### 评估总结
如图
![](./assets/图3评估总结.png)
