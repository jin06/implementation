#### q.Exec()


query的ng变量是一个Engine结构，所以调用的是engine的exec方法。

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

删除非关键性代码后，代码如下：

```golang
// execEvalStmt evaluates the expression of an evaluation statement for the given time range.
func (ng *Engine) execEvalStmt(ctx context.Context, query *query, s *parser.EvalStmt) (parser.Value, storage.Warnings, error) {
	prepareSpanTimer, ctxPrepare := query.stats.GetSpanTimer(ctx, stats.QueryPreparationTime, ng.metrics.queryPrepareTime)
	mint, maxt := ng.findMinMaxTime(s)
    // 这个位置返回了一个storage目录下的 Querier结构
	querier, err := query.queryable.Querier(ctxPrepare, mint, maxt)
	if err != nil {
		prepareSpanTimer.Finish()
		return nil, nil, err
	}
	defer querier.Close()

	ng.populateSeries(querier, s)
	prepareSpanTimer.Finish()

	// Modify the offset of vector and matrix selectors for the @ modifier
	// w.r.t. the start time since only 1 evaluation will be done on them.
	setOffsetForAtModifier(timeMilliseconds(s.Start), s.Expr)
	evalSpanTimer, ctxInnerEval := query.stats.GetSpanTimer(ctx, stats.InnerEvalTime, ng.metrics.queryInnerEval)
	// Instant evaluation. This is executed as a range evaluation with one step.
	if s.Start == s.End && s.Interval == 0 {
		start := timeMilliseconds(s.Start)
		evaluator := &evaluator{
			startTimestamp:           start,
			endTimestamp:             start,
			interval:                 1,
			ctx:                      ctxInnerEval,
			maxSamples:               ng.maxSamplesPerQuery,
			logger:                   ng.logger,
			lookbackDelta:            ng.lookbackDelta,
			samplesStats:             query.sampleStats,
			noStepSubqueryIntervalFn: ng.noStepSubqueryIntervalFn,
		}
		query.sampleStats.InitStepTracking(start, start, 1)

		val, warnings, err := evaluator.Eval(s.Expr)
		if err != nil {
			return nil, warnings, err
		}

		evalSpanTimer.Finish()

		var mat Matrix

		switch result := val.(type) {
		case Matrix:
			mat = result
		case String:
			return result, warnings, nil
		default:
			panic(fmt.Errorf("promql.Engine.exec: invalid expression type %q", val.Type()))
		}

		query.matrix = mat
		switch s.Expr.Type() {
		case parser.ValueTypeVector:
			// Convert matrix with one value per series into vector.
			vector := make(Vector, len(mat))
			for i, s := range mat {
				// Point might have a different timestamp, force it to the evaluation
				// timestamp as that is when we ran the evaluation.
				vector[i] = Sample{Metric: s.Metric, Point: Point{V: s.Points[0].V, T: start}}
			}
			return vector, warnings, nil
		case parser.ValueTypeScalar:
			return Scalar{V: mat[0].Points[0].V, T: start}, warnings, nil
		case parser.ValueTypeMatrix:
			return mat, warnings, nil
		default:
			panic(fmt.Errorf("promql.Engine.exec: unexpected expression type %q", s.Expr.Type()))
		}
	}

	// Range evaluation.
	evaluator := &evaluator{
		startTimestamp:           timeMilliseconds(s.Start),
		endTimestamp:             timeMilliseconds(s.End),
		interval:                 durationMilliseconds(s.Interval),
		ctx:                      ctxInnerEval,
		maxSamples:               ng.maxSamplesPerQuery,
		logger:                   ng.logger,
		lookbackDelta:            ng.lookbackDelta,
		samplesStats:             query.sampleStats,
		noStepSubqueryIntervalFn: ng.noStepSubqueryIntervalFn,
	}
	query.sampleStats.InitStepTracking(evaluator.startTimestamp, evaluator.endTimestamp, evaluator.interval)
	val, warnings, err := evaluator.Eval(s.Expr)
	if err != nil {
		return nil, warnings, err
	}
	evalSpanTimer.Finish()

	mat, ok := val.(Matrix)
	if !ok {
		panic(fmt.Errorf("promql.Engine.exec: invalid expression type %q", val.Type()))
	}
	query.matrix = mat

	if err := contextDone(ctx, "expression evaluation"); err != nil {
		return nil, warnings, err
	}

	// TODO(fabxc): where to ensure metric labels are a copy from the storage internals.
	sortSpanTimer, _ := query.stats.GetSpanTimer(ctx, stats.ResultSortTime, ng.metrics.queryResultSort)
	sort.Sort(mat)
	sortSpanTimer.Finish()

	return mat, warnings, nil
}
```