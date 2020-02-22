---
title: "try session"
date: 2020-02-09T22:35:57+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
林下漏月光，疏疏如残雪。——张岱《陶庵梦忆》
<!--more-->
# trySession
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build_session.go
func trySession(dockerCli command.Cli, contextDir string, forStream bool) (*session.Session, error) {
    if !isSessionSupported(dockerCli, forStream) {
        return nil, nil
    }
    sharedKey := getBuildSharedKey(contextDir)
    s, err := session.NewSession(context.Background(), filepath.Base(contextDir), sharedKey)
    if err != nil {
        return nil, errors.Wrap(err, "failed to create session")
    }
    return s, nil
}
{{< /highlight >}}

# isSessionSupported
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build_session.go
func isSessionSupported(dockerCli command.Cli, forStream bool) bool {
    if !forStream && versions.GreaterThanOrEqualTo(dockerCli.Client().ClientVersion(), "1.39") {
        return true
    }
    return dockerCli.ServerInfo().HasExperimental && versions.GreaterThanOrEqualTo(dockerCli.Client().ClientVersion(), "1.31")
}
{{< /highlight >}}

# getBuildSharedKey
getBuildSharedKey根据dir生成一个随机数
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build_session.go
func getBuildSharedKey(dir string) string {
    // build session is hash of build dir with node based randomness
    s := sha256.Sum256([]byte(fmt.Sprintf("%s:%s", tryNodeIdentifier(), dir)))
    return hex.EncodeToString(s[:])
}
{{< /highlight >}}

# tryNodeIdentifier
{{< highlight go "linenos=inline" >}}
func tryNodeIdentifier() string {
    out := cliconfig.Dir() // return config dir as default on permission error
    if err := os.MkdirAll(cliconfig.Dir(), 0700); err == nil {
        sessionFile := filepath.Join(cliconfig.Dir(), ".buildNodeID")
        if _, err := os.Lstat(sessionFile); err != nil {
            if os.IsNotExist(err) { // create a new file with stored randomness
                b := make([]byte, 32)
                if _, err := rand.Read(b); err != nil {
                    return out
                }
                if err := ioutil.WriteFile(sessionFile, []byte(hex.EncodeToString(b)), 0600); err != nil {
                    return out
                }
            }
        }

        dt, err := ioutil.ReadFile(sessionFile)
        if err == nil {
            return string(dt)
        }
    }
    return out
}
cli/com
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
