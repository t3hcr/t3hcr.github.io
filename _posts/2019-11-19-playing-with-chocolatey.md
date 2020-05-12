# Playing with Chocolatey

I started playing with [Chocolatey] today. I'm planning to use it to install and update software used by myself and fellow administrators on various Windows servers.  The nice thing about it is there's no "click, click, next, finish" to it.  It's repeatable and reproducible and makes admin life easier.

To top it off, when used in combination with PowerShell you can remotely install software.  As an example: 

```PowerShell
Invoke-Command -ComputerName HOST1,HOST2,HOST3 {
    choco install git.install -y;
    choco install sysinternals -y;
    choco install vscode -y;
    choco install vscode-powershell -y
} -Credential (Get-Credential) -Verbose
```

I'm just getting started with it and am curious to see how upgrading software works. 

Are you using Chocolatey? What are you using it for? I'd be curious to hear from others to learn from you. 😊

[Chocolatey]:https://www.chocolatey.org
