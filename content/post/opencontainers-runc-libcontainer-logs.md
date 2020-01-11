---
title: "opencontainers-runc-libcontainer-logs"
date: 2019-10-29T06:09:32+08:00
draft: true
categories: ["技术"]
tags: ["runc"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/opencontainers/runc/libcontainer/logs/logs.go
{{< /highlight >}}

package logs

# 变量
{{< highlight go "linenos=inline" >}}
var (
	configureMutex = sync.Mutex{}
	// loggingConfigured will be set once logging has been configured via invoking `ConfigureLogging`.
	// Subsequent invocations of `ConfigureLogging` would be no-op
	loggingConfigured = false
)
{{< /highlight >}}

# Config
{{< highlight go "linenos=inline" >}}
type Config struct {
	LogLevel    logrus.Level
	LogFormat   string
	LogFilePath string
	LogPipeFd   string
}
{{< /highlight >}}

# ForwardLogs
{{< highlight go "linenos=inline" >}}
func ForwardLogs(logPipe io.Reader) {
	lineReader := bufio.NewReader(logPipe)
	for {
		line, err := lineReader.ReadBytes('\n')
		if len(line) > 0 {
			processEntry(line)
		}
		if err == io.EOF {
			logrus.Debugf("log pipe has been closed: %+v", err)
			return
		}
		if err != nil {
			logrus.Errorf("log pipe read error: %+v", err)
		}
	}
}
{{< /highlight >}}

## 库函数说明
{{< highlight go "linenos=inline" >}}
import "bufio"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func NewReader(rd io.Reader) *Reader
{{< /highlight >}}

>  NewReader returns a new Reader whose buffer has the default size.  

{{< highlight go "linenos=inline" >}}
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
{{< /highlight >}}

> ReadBytes reads until the first occurrence of delim in the input, returning a slice containing the data up to and including the delimiter. If ReadBytes encounters an error before finding a delimiter, it returns the data read before the error and the error itself (often io.EOF). ReadBytes returns err != nil if and only if the returned data does not end in delim. For simple uses, a Scanner may be more convenient.

字符常量是用单引号('')括起来的单个字符，在golang中，ASCII表中的字符都可以直接赋值给byte类型变量，字符类型是可以进行运算的，相当于一个整数，因为它都对应有Unicode 码。

{{< highlight go "linenos=inline" >}}
var c1 byte = 'a'
var c2 byte = '0' // 字符0
var c3 char = '\n'
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func NewWriter(w io.Writer) *Writer
{{< /highlight >}}

> NewWriter returns a new Writer whose buffer has the default size.

{{< highlight go "linenos=inline" >}}
func NewScanner(r io.Reader) *Scanner
{{< /highlight >}}

> NewScanner returns a new Scanner to read from r. The split function defaults to ScanLines.


{{< highlight go "linenos=inline" >}}
func (s *Scanner) Buffer(buf []byte, max int)
{{< /highlight >}}

> Buffer sets the initial buffer to use when scanning and the maximum size of buffer that may be allocated during scanning. The maximum token size is the larger of max and cap(buf). If max <= cap(buf), Scan will use this buffer only and do no allocation.  
By default, Scan uses an internal buffer and sets the maximum token size to MaxScanTokenSize.  
Buffer panics if it is called after scanning has started.


# processEntry
{{< highlight go "linenos=inline" >}}
func processEntry(text []byte) {
	type jsonLog struct {
		Level string `json:"level"`
		Msg   string `json:"msg"`
	}

	var jl jsonLog
	if err := json.Unmarshal(text, &jl); err != nil {
		logrus.Errorf("failed to decode %q to json: %+v", text, err)
		return
	}

	lvl, err := logrus.ParseLevel(jl.Level)
	if err != nil {
		logrus.Errorf("failed to parse log level %q: %v\n", jl.Level, err)
		return
	}
	logrus.StandardLogger().Logf(lvl, jl.Msg)
}
{{< /highlight >}}

# ConfigureLogging
{{< highlight go "linenos=inline" >}}
func ConfigureLogging(config Config) error {
	configureMutex.Lock()
	defer configureMutex.Unlock()

	if loggingConfigured {
		logrus.Debug("logging has already been configured")
		return nil
	}

	logrus.SetLevel(config.LogLevel)

	if config.LogPipeFd != "" {
		logPipeFdInt, err := strconv.Atoi(config.LogPipeFd)
		if err != nil {
			return fmt.Errorf("failed to convert _LIBCONTAINER_LOGPIPE environment variable value %q to int: %v", config.LogPipeFd, err)
		}
		logrus.SetOutput(os.NewFile(uintptr(logPipeFdInt), "logpipe"))
	} else if config.LogFilePath != "" {
		f, err := os.OpenFile(config.LogFilePath, os.O_CREATE|os.O_WRONLY|os.O_APPEND|os.O_SYNC, 0644)
		if err != nil {
			return err
		}
		logrus.SetOutput(f)
	}

	switch config.LogFormat {
	case "text":
		// retain logrus's default.
	case "json":
		logrus.SetFormatter(new(logrus.JSONFormatter))
	default:
		return fmt.Errorf("unknown log-format %q", config.LogFormat)
	}

	loggingConfigured = true
	return nil
}
{{< /highlight >}}
