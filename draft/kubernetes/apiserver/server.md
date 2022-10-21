
### APIServer 注册handler

注册的逻辑在这里 vendor/k8s.io/apiserver/pkg/endpoints/installer.go:190

```golang
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
}
```

会将具体处理的接口先处理成 action，组成一个actions数组. action的定义在 vendor/k8s.io/apiserver/pkg/endpoints/installer.go:63

```golang
// Struct capturing information about an action ("GET", "POST", "WATCH", "PROXY", etc).
type action struct {
	Verb          string               // Verb identifying the action ("GET", "POST", "WATCH", "PROXY", etc).
	Path          string               // The path of the action
	Params        []*restful.Parameter // List of parameters associated with the action.
	Namer         handlers.ScopeNamer
	AllNamespaces bool // true iff the action is namespaced but works on aggregate result for all namespaces
}
```

然后遍历这个actions。遍历的时候根有一个switch语句，根据其中的Verb来具体处理，处理成route后放入routes。route的定义为, vendor/github.com/emicklei/go-restful/v3/route_builder.go:19

 ```golang
 // RouteBuilder is a helper to construct Routes.
type RouteBuilder struct {
	rootPath                         string
	currentPath                      string
	produces                         []string
	consumes                         []string
	httpMethod                       string        // required
	function                         RouteFunction // required
	filters                          []FilterFunction
	conditions                       []RouteSelectionConditionFunction
	allowedMethodsWithoutContentType []string // see Route

	typeNameHandleFunc TypeNameHandleFunction // required

	// documentation
	doc                     string
	notes                   string
	operation               string
	readSample, writeSample interface{}
	parameters              []*Parameter
	errorMap                map[int]ResponseError
	defaultResponse         *ResponseError
	metadata                map[string]interface{}
	extensions              map[string]interface{}
	deprecated              bool
	contentEncodingEnabled  *bool
}
 ```