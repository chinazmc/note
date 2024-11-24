gin中加个中间件，
```go
func MiddleWareAccessLogger() func(*Context) {  
   return func(ctx *Context) {  
  
      var alarmId string  
  
      var requestBody string  
  
      if !isMultipartForm(ctx) {  
         // 每次读取request body 会 清空buffer, 所以打印日志需要, 拷贝一份request body buffer。  
         bodyBs, _ := ioutil.ReadAll(ctx.Request.Body)  
         copyRequestBody := ioutil.NopCloser(bytes.NewBuffer(bodyBs))  
         // 拷贝buffer后重写入到request中。  
         ctx.Request.Body = copyRequestBody  
         requestBody = stringUtil.BytesToString(bodyBs)  
      } else {  
         requestBody = "file body"  
      }  
  
      param := fmt.Sprintf("Header: %s \nQueryString: %s \nBody: %s", formatHeader(ctx.Request.Header), formatQueryString(ctx.Request.URL.Query()), requestBody)  
  
      start := time.Now()  
      path := ctx.Request.URL.Path  
  
      alarmId = getTraceId(ctx)  
  
      defer func() {  
         if err := recover(); err != nil {  
            logger.DefaultSudaLogger.Error(  
               logger.LogInterface, ctx.Request.URL.Path,  
               logger.LogResult, fmt.Sprintf("panic : %v", errors.Wrap(err, 2).ErrorStack()),  
               logger.LogAlarmID, getTraceId(ctx),  
               logger.LogParam, param,  
            )  
            go dingtalk_robot.SendMsg(config.GetString("env"), config.GetString("name"), "panic", ctx.Request.URL.Path+" \n #### TraceID:"+getTraceId(ctx),  
               getTraceId(ctx), kibana.GetLoggerLink(config.GetString("env"), getTraceId(ctx)), true)  
  
            ctx.JSON(http.StatusOK, errcode.SYSTEM_ERROR.ToResult())  
         }  
      }()  
  
      blw := &bodyLogWriter{body: bytes.NewBufferString(""), ResponseWriter: ctx.Writer}  
      ctx.Writer = blw  
  
      ctx.Next()  
  
      result := blw.body.String()  
      statusCode := ctx.Writer.Header().Get("x-status")  
  
      if len(statusCode) == 0 {  
         statusCode = "9999"  
         result = "response body"  
      }  
  
      if doAccessLog(path) {  
         logger.DefaultSudaLogger.LogApiInfo(path, param, result, statusCode, fmt.Sprint(time.Now().Sub(start)), "", alarmId)  
      }  
      if DoStatisticsLog {  
         prometheus.RecordStatus(path, statusCode)  
      }  
   }  
}
```