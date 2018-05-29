# Running Kops in Docker

The Dockerfile here is offered primarily as a way to build continuous
integration versions of `kops` until we figure out how we want to
release/package it.

To use it, e.g. (assumes your `$HOME` is correct and that `$KOPS_STATE_STORE` is correct):
```shell
$ docker build -t kops .
$ KOPS="docker run -v $HOME/.aws:/root/.aws:ro -v $HOME/.ssh:/root/.ssh:ro -v $HOME/.kube:/root/.kube -it kops --state=$KOPS_STATE_STORE"
```

This creates a shell variable that runs the `kops` container with `~/.aws` mounted in (for AWS credentials), `~/.ssh` mounted in (for SSH keys, for AWS specifically), and `~/.kube` mounted in (so `kubectl` can add newly created clusters).

After this, you can just use `$KOPS` where you would generally use `kops`, e.g. `$KOPS get cluster`.

#### Choose branch/release to build.
By default, the current release branch is built.  To build using a specific tag or commit, add the flag `--build-arg KOPS_GITISH=<tag/branch/sha>` to `docker build`, e.g. `docker build --build-arg KOPS_GITISH=release-1.6 -t kops .`

#### Light Version
The light version downloads the latest release binaries of kops from [Github Releases](https://github.com/kubernetes/kops/releases).

To build the lighter version:
```shell
$ docker build -t kops:light -f Dockerfile-light .
$ KOPS="docker run -v $HOME/.aws:/root/.aws:ro -v $HOME/.ssh:/root/.ssh:ro -v $HOME/.kube:/root/.kube -it kops:light --state=$KOPS_STATE_STORE"
  ```
### Powershell wrapper
This powershell wrapper allows for kops to be used in Powershell. It assumes that the AWS SDK and Docker for Windows are installed and that the containermode for Docker is set to Linux.

```powershell
function Create-KopsContainer {
    $basePath = "$HOME\AppData\Local\Temp\KOPS\"
    if(!(Test-Path $basePath)) {
        md $basePath | Out-Null
    }
    $tempPath = Join-Path $basePath "Dockerfile-light"
    Invoke-WebRequest -Uri "https://raw.githubusercontent.com/kubernetes/kops/master/docker/Dockerfile-light" -OutFile $tempPath
    iex "docker build -t kops:light -f $tempPath ."
}

function Invoke-Kops {
    param (
        [Parameter(ValueFromRemainingArguments=$true)] $extras = $null
    )
    $dockerPath = get-command "docker.exe" | select -ExpandProperty source -first 1
    $kubePath = Join-Path $env:Home '/.kube'
    if(!(Test-Path $kubePath)) {
        md $kubePath
    }
    $cliArgs = @(
        "run",
        "--env AWS_SDK_LOAD_CONFIG=1",
        "-it",
        "-v $((Join-Path $env:Home '/.aws').Replace('\','/')):/root/.aws:ro",
        "-v $((Join-Path $env:Home '/.ssh').Replace('\','/')):/root/.ssh:ro",
        "-v $($kubePath.Replace('\','/')):/root/.kube",
        "kops:light"
    )
    if($env:KOPS_STATE_STORE) {
        $cliArgs += @("--state=$env:KOPS_STATE_STORE")
    }
    if($extras) {
        $cliArgs += $extras
    }

    Start-Process $dockerPath -NoNewWindow -ArgumentList $cliArgs -Wait
}

Set-Alias -Name KOPS -Value Invoke-Kops
```
