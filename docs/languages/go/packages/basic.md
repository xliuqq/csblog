# 基础包

## net



### pprof

golang 提供的 pprof 工具可以很方便的分析性能上的问题比如cpu的使用情况，堆内存分配，goroutine 死锁情况等。

```go
mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)

pprofServer := http.Server{
    Addr:    pprofAddr,
    Handler: mux,
}
setupLog.Info("Starting pprof HTTP server", "pprof server address", pprofServer.Addr)

go func() {
    go func() {
        ctx := context.Background()
        <-ctx.Done()

        ctx, cancelFunc := context.WithTimeout(context.Background(), 60*time.Minute)
        defer cancelFunc()

        if err := pprofServer.Shutdown(ctx); err != nil {
            setupLog.Error(err, "Failed to shutdown debug HTTP server")
        }
    }()

    if err := pprofServer.ListenAndServe(); !errors.Is(http.ErrServerClosed, err) {
        setupLog.Error(err, "Failed to start debug HTTP server")
        panic(err)
    }
}()
```



## json

json包是通过反射机制来实现编解码的，因此结构体必须导出所转换的字段，不导出的字段不会被json包解析。

 ```go
 type DebugInfo struct {
 	Level  string `json:"level,omitempty"` // Level解析为level,忽略空值
 	Msg    string `json:"message"`     // Msg解析为message
 	Author string `json:"-"`        // 忽略Author
 }
 ```



## log 

### Zap logging

> [zap](https://github.com/uber-go/zap)是uber开源的一款高性能日志组件框架



### github.com/golang/glog



### github.com/go-logr/logr

> A simple logging interface for Go

Log 门面实现（类似slf4j）

```go
log.V(1).Info("job count", "active jobs", len(activeJobs), "successful jobs", len(successfulJobs), "failed jobs", len(failedJobs))
```



#### github.com/go-logr/glogr 

> An implementation of logr (Go logging) with glog

#### github.com/go-logr/zapr

> A logr implementation using Zap

**使用**

```
import (
    "fmt"

    "go.uber.org/zap"
    "github.com/go-logr/logr"
    "github.com/go-logr/zapr"
)

func main() {
    var log logr.Logger

    zapLog, err := zap.NewDevelopment()
    if err != nil {
        panic(fmt.Sprintf("who watches the watchmen (%v)?", err))
    }
    log = zapr.NewLogger(zapLog)

    log.Info("Logr in action!", "the answer", 42)
}
```





## CLI（Cobra）

> [CLI](https://github.com/spf13/cobra)：A Commander for modern Go CLI interactions

- Easy subcommand-based CLIs: `app server`, `app fetch`, etc.
- Fully POSIX-compliant flags (including short & long versions)
- Nested subcommands
- Global, local and cascading flags
- Easy generation of applications & commands with `cobra init` & `cobra add cmdname`
- Intelligent suggestions (`app srver`... did you mean `app server`?)
- Automatic help generation for commands and flags
- Automatic help flag recognition of `-h`, `--help`, etc.
- Automatically generated shell autocomplete for your application (bash, zsh, fish, powershell)
- Automatically generated man pages for your application
- Command aliases so you can change things without breaking them
- The flexibility to define your own help, usage, etc.
- Optional seamless integration with [viper](http://github.com/spf13/viper) for 12-factor apps

### 原理

**Commands** represent actions

**Args** are things 

**Flags** are modifiers for those actions.

```shell
# server is a command, and port is a flag
$ hugo server --port=1313
```



### demo

```go
import ( "github.com/spf13/cobra" )
var (
	short                bool
	metricsAddr          string
	enableLeaderElection bool
	development          bool
	pprofAddr            string
)
var cmd = &cobra.Command{
	Use:   "dataset-controller",
	Short: "controller for dataset",
}
var startCmd = &cobra.Command{
	Use:   "start",
	Short: "start dataset-controller in Kubernetes",
	Run: func(cmd *cobra.Command, args []string) {
		handle()
	}
}

func init() {
  	startCmd.Flags().StringVarP(&metricsAddr, "metrics-addr", "", ":8080", "The address the metric endpoint binds to.")
	startCmd.Flags().BoolVarP(&enableLeaderElection, "enable-leader-election", "", false, "Enable leader election for controller manager. Enabling this will ensure there is only one active controller manager.")
	startCmd.Flags().BoolVarP(&development, "development", "", true, "Enable development mode for fluid controller.")
	startCmd.Flags().StringVarP(&pprofAddr, "pprof-addr", "", "", "The address for pprof to use while exporting profiling results")
	cmd.AddCommand(startCmd)
}

func main() {
    if err := cmd.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%s", err.Error())
		os.Exit(1)
	}
}
```



## Template

https://golang.google.cn/pkg/text/template/

模板引擎



## [viper](https://github.com/spf13/viper/)（配置读写）

`go get github.com/spf13/viper`

### 读

```
viper.SetConfigName("config") // name of config file (without extension)
viper.SetConfigType("yaml") // REQUIRED if the config file does not have the extension in the name
viper.AddConfigPath("/etc/appname/")   // path to look for the config file in
viper.AddConfigPath("$HOME/.appname")  // call multiple times to add many search paths
viper.AddConfigPath(".")               // optionally look for config in the working directory
err := viper.ReadInConfig() // Find and read the config file
if err != nil { // Handle errors reading the config file
	panic(fmt.Errorf("fatal error config file: %w", err))
}
```
