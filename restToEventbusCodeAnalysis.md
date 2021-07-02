##  rest到eventbus过程分析

1. 需要先部署 rest 和 eventbus的RuleEndpoints文件

   根据上一篇对router的分析，部署了RuleEndpints之后，RuleEndpoints集合内容为：

   ruleEndpoints["default/my-rest"] = my-rest 的ruleEndpoints对象

   ruleEndpoints["default/my-eventbus"] = my-eventbus的ruleEndpoints对象

```yaml
apiVersion: rules.kubeedge.io/v1
kind: RuleEndpoint
metadata:
  name: my-rest
  labels:
    description: test
spec:
  ruleEndpointType: "rest"
  properties: {}
```

```yaml
apiVersion: rules.kubeedge.io/v1
kind: RuleEndpoint
metadata:
  name: my-eventbus
  labels:
    description: test
spec:
  ruleEndpointType: "eventbus"
  properties: {}
```



2. 部署rule文件

   ```yaml
   apiVersion: rules.kubeedge.io/v1
   kind: Rule
   metadata:
     name: my-rule
     labels:
       description: test
   spec:
     source: "my-rest"
     sourceResource: {"path":"/test"}
     target: "my-eventbus"
     targetResource: {"topic":"test"}
   ```

   ```go
   // AddRule add rule
   func addRule(rule *routerv1.Rule) error {
   	source, err := getSourceOfRule(rule)
   	...
   	target, err := getTargetOfRule(rule)
   	...
   	ruleKey := getKey(rule.Namespace, rule.Name)
   	if err := source.RegisterListener(func(data interface{}) (interface{}, error) {
   		//TODO Use goroutine pool later
   		var execResult ExecResult
   		resp, err := source.Forward(target, data) //转发数据
   		...
   	}); err != nil {
   		...
   	}
   	rules.Store(ruleKey, rule)
   	klog.Infof("add rule success: %+v", rule)
   	return nil
   }
   ```

   ```golang
   func (r *Rest) RegisterListener(handle listener.Handle) error {
   	listener.RestHandlerInstance.AddListener(fmt.Sprintf("/%s/%s", r.Namespace, r.Path), handle)
   	return nil
   }
   ```

   根据上篇对router的分析：添加rule之后，source对象，即rest对象会增加一个监听函数，路由为/default/test的消息，交给handle处理，handle内容主要为，将消息转发到target。 

   **note** ：getSourceOfRule函数，source为rest对象时，他会检查现在是否有启动一个server服务器，如果没有他启动一个服务器，端口默认为9443

   ```go
   
   func getSourceOfRule(rule *routerv1.Rule) (provider.Source, error) {
     ....
   	sf, exist := provider.GetSourceFactory(sourceEp.Spec.RuleEndpointType)
     ....
   	source := sf.GetSource(sourceEp, rule.Spec.SourceResource)
   	return source, nil
   }
   func (*restFactory) GetSource(ep *v1.RuleEndpoint, sourceResource map[string]string) provider.Source {
   	path, exist := sourceResource["path"]
     .....
   	cli := &Rest{Namespace: ep.Namespace, Path: normalizeResource(path)}
   	if atomic.CompareAndSwapInt32(&inited, 0, 1) {   //
   		listener.InitHandler()
   		// guarantee that it will be executed only once
   		go listener.RestHandlerInstance.Serve()
   	}
   	return cli
   }
   ```

   forward函数如下所示：

   ```go
   func (r *Rest) Forward(target provider.Target, data interface{}) (interface{}, error) {
   	d := data.(map[string]interface{})
   	v, exist := d["request"]
     ....
   	request, ok := v.(*http.Request)
   	....
   	res := make(map[string]interface{})
   	messageID := d["messageID"].(string)
   	res["messageID"] = messageID
   	res["param"] = strings.TrimLeft(uri[3], r.Path)
   	res["data"] = d["data"]
   	res["nodeName"] = strings.Split(request.RequestURI, "/")[1]
   	res["header"] = request.Header
   	res["method"] = request.Method
   	stop := make(chan struct{})
   	respch := make(chan interface{})
   	errch := make(chan error)
   	go func() {
   		resp, err := target.GoToTarget(res, stop)   // 重点，eventbus的GoToTarget函数
   		if err != nil {
   			errch <- err
   			return
   		}
   		respch <- resp
   	}()
   	timer := time.NewTimer(timeout)
   	var httpResponse = &http.Response{
   		Request: request,
   		Header:  http.Header{},
   	}
   	..... 
   	return httpResponse, nil
   }
   ```

3. 转发到cloudhub模块

   ```go
   
   func (eb *EventBus) GoToTarget(data map[string]interface{}, stop chan struct{}) (interface{}, error) {
   	messageID, ok := data["messageID"].(string)
   	body, ok := data["data"].([]byte)
   	param, ok := data["param"].(string)
   	nodeName, ok := data["nodeName"].(string)
   	...
   	msg := model.NewMessage("")
   	msg.BuildHeader(messageID, "", msg.GetTimestamp())
   	resource := "node/" + nodeName + "/"   // 根据nodename转发到对应的节点
   	if !ok || param == "" {
   		resource = resource + eb.pubTopic
   	} else {
   		resource = resource + strings.TrimSuffix(eb.pubTopic, "/") + "/" + strings.TrimPrefix(param, "/")
   	}
   	msg.SetResourceOperation(resource, publishOperation) // publishOperation="publish"
   	msg.FillBody(string(body))
   	msg.SetRoute("router_eventbus", modules.UserGroup)
   	beehiveContext.Send(modules.CloudHubModuleName, *msg) // 把消息转发到cloudhub
   	return nil, nil
   }
   ```



4. edgehub模块把消息转发给eventbus，根据消息的operation，进入函数eb.publish,通过mqtt 发布这个消息。有订阅这个主题的mqtt客户端就会接受到这个消息。

```go
func (eb *eventbus) pubCloudMsgToEdge() {
	for {
    ...
		accessInfo, err := beehiveContext.Receive(eb.Name())
    
		operation := accessInfo.GetOperation()
		resource := accessInfo.GetResource()
		switch operation {
      .....
		case messagepkg.OperationPublish:
			topic := resource
			// cloud and edge will send different type of content, need to check
			payload, ok := accessInfo.GetContent().([]byte)
			if !ok {
				content, ok := accessInfo.GetContent().(string)
				if !ok {
					klog.Errorf("Message is not []byte or string")
					continue
				}
				payload = []byte(content)
			}
			eb.publish(topic, payload)
      ....
		}
	}
}
```

5. publish函数

   ```go
   func (eb *eventbus) publish(topic string, payload []byte) {
   	if eventconfig.Config.MqttMode >= v1alpha1.MqttModeBoth {
   		// pub msg to external mqtt broker.
   		pubMQTT(topic, payload)
   	}
   
   	if eventconfig.Config.MqttMode <= v1alpha1.MqttModeBoth {
   		// pub msg to internal mqtt broker.
   		mqttServer.Publish(topic, payload)
   	}
   }
   ```

   