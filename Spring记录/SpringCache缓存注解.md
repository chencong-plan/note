## `Spring缓存`使用记录

> 由于本次业务需求当中需要对支付渠道的异常码映射进行缓存操作，项目采用Redis作为NoSQL，客户端采用Redisson，SpringCache对其进行管理。故在此学习SpringCache当中如何使用注解，无侵入式对数据进行缓存操作。

在使用之前SpringCache之前，我们需要了解几件事情：① SpringCache是作用在方法上的，其核心思想为：当我们调用一个方法(get or set)时，如若在其方法上存在注解，会将方法的参数和返回结果作为键值对形式存储在缓存中，当下一下同样参数(key)将从缓存当中获取，不执行方法。② 使用SpringCache时需要保证缓存的方法对于相同的方法参数要有相同的返回结果。准备一下内容：

+ 思考将要对哪些方法使用缓存
+ 配置SpringCache支持。同样支持注解形式和XML两种配置方式。

### 基于注解形式

在使用Redis时，我们常用set 和 get 对键和值进行操作。类比在SpringCache当中我们使用`@Cacheable`和`@CacheEvict`两个注解进行标记方法。前者表示：在执行方法时，先从缓存中获取该对应key（可配置）的值，如果查询得到，则直接返回不执行方法；如果没有，则执行方法，并将返回结果放入缓存当中。`@CacheEvict`表示执行标记的方法前，执行该注解将缓存中对应数据进行移除。

#### @Cacheable

+ 可以标记类和方法。在类上表示该类中所有方法支持缓存，在方法上表示该方法支持缓存。

+ 对于方法，表示spring会将其调用结果后的返回值缓存起来，以保证下次同样的参数执行该方法时可以直接从缓存当中获取，而不用执行该方法。

+ 缓存使用键值对，键：默认策略和自定义策略；值：方法返回值。

+ @Cacheable 具备三个参数：key value condition

  > ```java
  > @Cacheable(value="TsysResp",key="T(String).valueOf(#thirdType).concat('').concat(#thirdRespCode)")
  > 	public TsysResp getSysRespByTypeAndCode(String thirdType, String thirdRespCode) {
  > 		TsysRespExample example = new TsysRespExample();
  > 		example.createCriteria().andThirdTypeEqualTo(thirdType).andThirdRespCodeEqualTo(thirdRespCode);
  > 		List<TsysResp> tsysResps = tsysRespMapper.selectByExample(example);
  > 		return tsysResps.isEmpty() ? null : tsysResps.get(0);
  > 	}
  > ```

  此块代码使用@Cacheable,表示将方法缓存在`TsysResp`块上（可以任意修改，根据自己实际情况），`key`表示缓存键的策略（此时我是thirdType和thirdRespCode两个参数拼接结果作为key）

  > ```java
  > 	@Cacheable(value= {"TsysParameters","TsysParameters2"}, key="#paramName")
  > 	public TsysParameters getParamByParamName(String paramName) {
  > 		TsysParametersExample example = new TsysParametersExample();
  > 		example.createCriteria().andParamNameEqualTo(paramName);
  > 		List<TsysParameters> list = tsysParametersMapper.selectByExample(example);
  > 		if(list==null || list.isEmpty()){
  > 			return null;
  > 		}else if(list.size() != 1){
  > 			throw new PlatformException(PlatformErrorCode.UNKNOWN_ERROR,"数据插入异常");
  > 		}
  > 		return list.get(0);
  > 	}
  > ```

  第一行注解表示该缓存的value同时存在于`TsysParamters`和`TsysParamters2`上。

除了上述使用方法参数作为key之外，Spring还为我们提供了一个root对象可以用来生成key。通过该root对象我们可以获取到以下信息。 

| 属性名称    | 描述                        | 示例                 |
| ----------- | --------------------------- | -------------------- |
| methodName  | 当前方法名                  | #root.methodName     |
| method      | 当前方法                    | #root.method.name    |
| target      | 当前被调用的对象            | #root.target         |
| targetClass | 当前被调用的对象的Class     | #root.targetClass    |
| args        | 当前方法参数组成的数组      | #root.args[0]        |
| caches      | 当前被调用的方法使用的Cache | #root.caches[0].name |

当我们要使用root对象的属性作为key时我们也可以将“#root”省略，因为Spring默认使用的就是root对象的属性。如：

```java
 @Cacheable(value={"users", "xxx"}, key="caches[1].name")
   public User find(User user) {
      returnnull;
   }

```

`condition`属性指定发生的条件

有的时候我们可能并不希望缓存一个方法所有的返回结果。通过condition属性可以实现这一功能。condition属性默认为空，表示将缓存所有的调用情形。其值是通过SpringEL表达式来指定的，当为true时表示进行缓存处理；当为false时表示不进行缓存处理，即每次调用该方法时该方法都会执行一次。如下示例表示只有当user的id为偶数时才会进行缓存。 

```java
@Cacheable(value={"users"}, key="#user.id", condition="#user.id%2==0")
   public User find(User user) {
      System.out.println("find user by user " + user);
      return user;
   }
```

