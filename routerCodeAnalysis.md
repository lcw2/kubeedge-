## router  源码分析

此文章是对router模块的大致分析，只分析了addRuleEndpoints 和 addRule过程。后面会有文章分析rest-servicebus， rest-eventbus的具体过程。

### 数据结构 sources - tartgets

addRule的时候会把自定义的eventbus，servicebus，rest加入到sources和targets集合中。

代码入口：cloud/pkg/router/provider/source.go

source.go 定义了 SourceFactory 和  source 接口。 在main函数执行之前初始化sources 集合。

```golang
var (
	// Modules map
	sources map[string]SourceFactory
)
func init() {
	sources = make(map[string]SourceFactory)
}
type SourceFactory interface {
	Type() string
	GetSource(ep *v1.RuleEndpoint, sourceResource map[string]string) Source
}
type Source interface {
	Name() string
	RegisterListener(handle listener.Handle) error
	UnregisterListener()
	Forward(Target, interface{}) (interface{}, error)
}

```

代码入口： cloud/pkg/router/provider/target.go

target.go 定义了 TargetFactory 和 Target接口，在main函数执行之前初始化targets集合

```golang
func init() {
	targets = make(map[string]TargetFactory)
}

type TargetFactory interface {
	Type() string
	GetTarget(ep *v1.RuleEndpoint, targetResource map[string]string) Target
}

type Target interface {
	Name() string
	GoToTarget(data map[string]interface{}, stop chan struct{}) (interface{}, error)
}
```



### 添加RuleEndPoint 和 Rule 过程

```golang
// 包变量
var (
	rules         sync.Map
	ruleEndpoints sync.Map
)
```

1. 准备工作

   kubeedge 初始了 registerListerner 函数，会监听endpoind和rule的变化

   代码入口：cloud/pkg/router/rule/rule.go

   endpointKey = "EdgeController/ruleendpoint"

   rulekey = “EdgeController/rule" 

   当接收到source为edgecontroller，resource=ruleendpoint的消息时，消息会交给handleRuleEndpoint处理。

   当消息的resource为rule时，消息会交给handleRule处理。

   ```golang
   func init() {
   	registerListener()
   }
   
   func registerListener() {
   	endpointKey := fmt.Sprintf("%s/%s", modules.EdgeControllerModuleName, model.ResourceTypeRuleEndpoint)
   	listener.MessageHandlerInstance.AddListener(endpointKey, handleRuleEndpoint)
   
   	ruleKey := fmt.Sprintf("%s/%s", modules.EdgeControllerModuleName, model.ResourceTypeRule)
   	listener.MessageHandlerInstance.AddListener(ruleKey, handleRule)
   }
   ```

2. handleRuleEndpoint 函数

   ```golang
   // implement listener.Handle
   func handleRuleEndpoint(data interface{}) (interface{}, error) {
     message,ok := data.(*model.Message)
     ....
   	switch message.GetOperation() {
   	case model.InsertOperation:
   		addRuleEndpoint(ruleEndpoint)   // 如果操作为插入，转到addRuleEndpoint函数
   	case model.DeleteOperation:
   		deleteRuleEndpoint(ruleEndpoint.Namespace, ruleEndpoint.Name) //如果操作为删除，转到deleteRuleEndppoint函数
   	default:
   		klog.Warningf("invalid message operation.")
   	}
   	return nil, nil
   }
   ```

   addRuleEndpoint： 就是把ruleEndpoint对象加入到ruleEndpoints集合中。同理，deleteRuleEndpoint函数就是从集合中删除

   ```golang
   func addRuleEndpoint(ruleEndpoint *routerv1.RuleEndpoint) {
   	key := getKey(ruleEndpoint.Namespace, ruleEndpoint.Name)
   	ruleEndpoints.Store(key, ruleEndpoint)
   	klog.Infof("add ruleendpoint %s success.", key)
   }
   ```

3. handleRule函数 ： 只有两个逻辑添加和删除，

   ```golang
   
   // implement listener.Handle
   func handleRule(data interface{}) (interface{}, error) {
   	message, ok := data.(*model.Message)
   	....
   	switch message.GetOperation() {
   	case model.InsertOperation:
   		addRuleWithRetry(rule) （添加失败的话，会每隔5s，重试3次）
   	case model.DeleteOperation:
   		delRule(rule.Namespace, rule.Name)
   	default:
   		klog.Warningf("invalid message operation.")
   	}
   	return nil, nil
   }
   ```

   addRule 函数：

   通过getSourceOfRule获取到rule yaml文件中的source 的sourceKey，在ruleEndpoints集合中获取具体的ruleEndpoints，并根据ruleEndpoints和sourceResource获取到具体source的结构体。

   getTargetOfRule同理。

   ```golang
   // AddRule add rule
   func addRule(rule *routerv1.Rule) error {
   	source, err := getSourceOfRule(rule) //获取source对象
   	...
   	target, err := getTargetOfRule(rule) //获取target对象
   	...
   	ruleKey := getKey(rule.Namespace, rule.Name)
   	if err := source.RegisterListener(func(data interface{}) (interface{}, error) { // 注册handler，当接收到消息，进行转发
   		//TODO Use goroutine pool later
   		var execResult ExecResult
   		resp, err := source.Forward(target, data) // 重点
   		if err != nil {
   			// rule.Status.Fail++
   			// record error info for rule
   			errMsg := ErrorMsg{Detail: err.Error(), Timestamp: time.Now()}
   			execResult = ExecResult{RuleID: rule.Name, ProjectID: rule.Namespace, Status: "FAIL", Error: errMsg}
   		} else {
   			execResult = ExecResult{RuleID: rule.Name, ProjectID: rule.Namespace, Status: "SUCCESS"}
   		}
   		ResultChannel <- execResult
   		return resp, nil
   	}); err != nil {
   		klog.Errorf("add rule %s failed, err: %v", ruleKey, err)
   		errMsg := ErrorMsg{Detail: err.Error(), Timestamp: time.Now()}
   		execResult := ExecResult{RuleID: rule.Name, ProjectID: rule.Namespace, Status: "FAIL", Error: errMsg}
   		ResultChannel <- execResult
   		return nil
   	}
   
   	rules.Store(ruleKey, rule)
   	klog.Infof("add rule success: %+v", rule)
   	return nil
   }
   ```

   删除rule规则

   ```golang
   // DelRule delete rule by rule id
   func delRule(namespace, name string) {
   	ruleKey := getKey(namespace, name) 
   	v, exist := rules.Load(ruleKey) // 获取rule对象
   	....
   	rule := v.(*routerv1.Rule)
   	source, err := getSourceOfRule(rule) // 获取source
   	...
   	source.UnregisterListener() // source对消息取消监听，把handler从集合中删除
   
   	rules.Delete(ruleKey)
   	klog.Infof("delete rule success: %s", ruleKey)
   }
   ```

   

