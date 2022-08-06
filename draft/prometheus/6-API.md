
API和web控制台的启动在同一个函数中，main函数启动prometheus server时会调用web相关代码。调用的代码位置在 web/web.go:268

func New(logger log.Logger, o *Options) *Handler {}

这个方法返回的是一个是*Handler结构，prometheus自己定义的，然后这个结构里有一个成员叫api，这个api就是api。因为它的api还包括了前端页面使用的一些api。

然后在这个Handler启动的时候调用api这个成员的register的方法。。。。。这一段比较绕，也没必要细看。总之最后提供了一系列的api。

这些api的定义在位置 web/api/v1/api.go:284， 其中有两个闭包函数，wrap , wrapAgent用来实现http middleware的功能。。。

wrap的作用：
- api跨域，在header中加入参数。。
- 检查TSDB是否就绪

wrapAgent 判断自己是不是agent，如果是，那么api统一返回错误？

```golang
func (api *API) Register(r *route.Router) {
	wrap := func(f apiFunc) http.HandlerFunc {
		hf := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			httputil.SetCORS(w, api.CORSOrigin, r)
			result := setUnavailStatusOnTSDBNotReady(f(r))
			if result.finalizer != nil {
				defer result.finalizer()
			}
			if result.err != nil {
				api.respondError(w, result.err, result.data)
				return
			}

			if result.data != nil {
				api.respond(w, result.data, result.warnings)
				return
			}
			w.WriteHeader(http.StatusNoContent)
		})
		return api.ready(httputil.CompressionHandler{
			Handler: hf,
		}.ServeHTTP)
	}

	wrapAgent := func(f apiFunc) http.HandlerFunc {
		return wrap(func(r *http.Request) apiFuncResult {
			if api.isAgent {
				return apiFuncResult{nil, &apiError{errorExec, errors.New("unavailable with Prometheus Agent")}, nil, nil}
			}
			return f(r)
		})
	}

	r.Options("/*path", wrap(api.options))

	r.Get("/query", wrapAgent(api.query))
    .....
```

