---
title: my-config
toc: true
date: 2021-09-26 14:01:30
# thumbnail: /images/2021/岁月的童话2.jpg
tags:
  - other
  - blog
categories:
  - other

---



<!--more-->


```sh


alias workspace='cd /Users/thh/workspace'
alias sha='shasum -a 256 '
alias untar='tar -zxvf '
alias vi='vim'
alias wget='wget -c '
alias ipi='ifconfig getifaddr en0'
alias ipe='curl ipinfo.io/ip'
alias c='clear'
alias cls='clear'
alias ipe='curl ipinfo.io/ip'
alias ipi='ifconfig getifaddr en0'
alias ping='ping -c 5'
alias sha='shasum -a 256 '
#alias phpstan='~/tools/vendor/bin/phpstan'
alias rphpstan='phpstan analyse --error-format table -c ~/tools/phpstan.neon'
alias needpwd=' openssl rand -base64 16'
alias showip='cat /etc/resolv.conf'
hello(){
        echo "\n"
        echo "needpwd\n"
        echo "ii\n"
        echo "showip\n"
        echo "gitcbr\n"
}
gitcbr()
{
    $1
    git checkout -b $1 origin/$1
}
gitpp()
{
        $1
        git push origin $1:$1
}

alias upsource="source ~/.zshrc"
alias hosts='code /etc/hosts'
alias phpunit='./vendor/bin/phpunit'

cddr()
{
        echo $1
        case $1 in
        php71 )
        docker exec -it  php-71 /bin/zsh
        ;;
        p71 )
        docker exec -it  php-71 /bin/zsh
        ;;
        php72 )
        docker exec -it  php-72 /bin/zsh
        ;;
        p72 )
        docker exec -it  php-72 /bin/zsh
        ;;
        php )
        docker exec -it  php-super /bin/zsh
        ;;
        p )
        docker exec -it  php-super /bin/zsh
        ;;
        nginx)
        docker exec -it  nginx /bin/sh
        ;;
        ng)
        docker exec -it  nginx /bin/sh
        ;;
        php8)
        docker exec -it  php-80 /bin/zsh
        ;;
        p8)
        docker exec -it  php-80 /bin/zsh
        ;;
        n)
        docker exec -it  nginx bash
        ;;
        esac
}

upngconfig(){
    docker exec nginx nginx -s reload
}


pdate()
{
         php -r 'echo date("Y-m-d H:i:s",time()).PHP_EOL;'
}

ptime()
{
  php -r 'echo time();'
}

source ~/.bash_profile

gitsetp()
{
        git config --global http.proxy 'http://127.0.0.1:11000'
        git config --global https.proxy 'http://127.0.0.1:11000'
}

gitunsetp()
{
        git config --global --unset http.proxy
        git config --global --unset https.proxy
}

# export PATH=$PATH:/Users/thh/go/bin
# export PATH=$PATH:/Users/thh/.local/platform-tools
# export PATH=$PATH:/Users/thh/.local/bin
# export MAVEN_HOME=/Users/thh/.local/apache-maven-3.8.1
# export PATH=$PATH:$MAVEN_HOME/bin
# export PATH="/opt/homebrew/opt/php@7.4/bin:$PATH"
# export PATH="/opt/homebrew/opt/php@7.4/sbin:$PATH"
# export PATH="/opt/homebrew/opt/node@12/bin:$PATH"
```
