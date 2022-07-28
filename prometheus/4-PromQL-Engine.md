PromQL Engine 在prometheus main函数中的启动代码。 cmd/prometheus/main.go

```golang
func main() {
    ...
    	if !agentMode {
            ...
            queryEngine = promql.NewEngine(opts)
            ruleManager = rules.NewManager(&rules.ManagerOptions{
                ...
                QueryFunc:       rules.EngineQueryFunc(queryEngine, fanoutStorage),
                ...
            })
            ...
            ...
        }
    ...
}
```

promql/engine.go:290 NewEngine() 生成一个Engine。将非关键性代码删除后： 

其中 
- ActiveQueryTracker: promql.NewActiveQueryTracker(localStoragePath, cfg.queryConcurrency, log.With(logger, "component", "activeQueryTracker"))
- NoStepSubqueryIntervalFn: noStepSubqueryInterval.Get,



```golang
// NewEngine returns a new engine.
func NewEngine(opts EngineOpts) *Engine {
	if opts.Logger == nil {
		opts.Logger = log.NewNopLogger()
	}

    ...

	if opts.LookbackDelta == 0 {
		opts.LookbackDelta = defaultLookbackDelta
		if l := opts.Logger; l != nil {
			level.Debug(l).Log("msg", "Lookback delta is zero, setting to default value", "value", defaultLookbackDelta)
		}
	}

    ...

	return &Engine{
		timeout:                  opts.Timeout,
		logger:                   opts.Logger,
		metrics:                  metrics,
		maxSamplesPerQuery:       opts.MaxSamples,
		activeQueryTracker:       opts.ActiveQueryTracker,
		lookbackDelta:            opts.LookbackDelta,
		noStepSubqueryIntervalFn: opts.NoStepSubqueryIntervalFn,
		enableAtModifier:         opts.EnableAtModifier,
		enableNegativeOffset:     opts.EnableNegativeOffset,
		enablePerStepStats:       opts.EnablePerStepStats,
	}
}
```

Engine 结构有两个对外的方法。分别是 
-  func (ng *Engine) NewInstantQuery(q storage.Queryable, opts *QueryOpts, qs string, ts time.Time) (Query, error) 
-  func (ng *Engine) NewRangeQuery(q storage.Queryable, opts *QueryOpts, qs string, start, end time.Time, interval time.Duration) (Query, error) 

两个方法都是初始化Query出来。看一下Query的定义,Query是一个接口。主要的方法是 Exec和Statement。它的实现只有一个，在同一个文件内的query结构。 
query 中的stmt应该是做解析language的。

```golang
type Query interface {
	// Exec processes the query. Can only be called once.
	Exec(ctx context.Context) *Result
	// Close recovers memory used by the query result.
	Close()
	// Statement returns the parsed statement of the query.
	Statement() parser.Statement
	// Stats returns statistics about the lifetime of the query.
	Stats() *stats.Statistics
	// Cancel signals that a running query execution should be aborted.
	Cancel()
	// String returns the original query string.
	String() string
}

// query implements the Query interface.
type query struct {
	// Underlying data provider.
	queryable storage.Queryable
	// The original query string.
	q string
	// Statement of the parsed query.
	stmt parser.Statement
	// Timer stats for the query execution.
	stats *stats.QueryTimers
	// Sample stats for the query execution.
	sampleStats *stats.QuerySamples
	// Result matrix for reuse.
	matrix Matrix
	// Cancellation function for the query.
	cancel func()

	// The engine against which the query is executed.
	ng *Engine
}

```

Engine本身有一个方法是初始化这个query的，是newQuery ， 位置在promql/engine.go:430
其中 query 的stmt对应的实例是

```golang 
	es := &parser.EvalStmt{
		Expr:     PreprocessExpr(expr, start, end),
		Start:    start,
		End:      end,
		Interval: interval,
	}
```

promql/parser/ast.go:58
```golang
// EvalStmt holds an expression and information on the range it should
// be evaluated on.
type EvalStmt struct {
	Expr Expr // Expression to be evaluated.

	// The time boundaries for the evaluation. If Start equals End an instant
	// is evaluated.
	Start, End time.Time
	// Time between two evaluated instants for the range [Start:End].
	Interval time.Duration
}
```

#### q.Exec()


这个方法如下，最终调用的是engine的exec方法。

```golang
// Exec implements the Query interface.
func (q *query) Exec(ctx context.Context) *Result {
	if span := trace.SpanFromContext(ctx); span != nil {
		span.SetAttributes(attribute.String(queryTag, q.stmt.String()))
	}

	// Exec query.
	res, warnings, err := q.ng.exec(ctx, q)

	return &Result{Err: err, Value: res, Warnings: warnings}
}
```

Engine 的 exec方法最终会调用 promql/engine.go:612

func (ng *Engine) execEvalStmt(ctx context.Context, query *query, s *parser.EvalStmt) (parser.Value, storage.Warnings, error)
