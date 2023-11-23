PROJECT_ROOT:=/home/isucon/webapp
BUILD_DIR:=go
# BIN_NAME:=
BRANCH:=main

NGX_LOG:=/tmp/access.log
MYSQL_LOG:=/tmp/slow-query.log

DB_HOST:=127.0.0.1
DB_PORT:=3306
DB_USER:=isucon
DB_PASS:=isucon
DB_NAME:=isuports
MYSQL_CMD:=mysql -h$(DB_HOST) -P$(DB_PORT) -u$(DB_USER) -p$(DB_PASS) $(DB_NAME)

SERVICE_NAME:=isuports

ALPSORT=sum
ALPM="/api/organizer/players/[0-9a-zA-Z]+/disqualified,/api/organizer/competition/[0-9a-zA-Z]+/finish,/api/organizer/competition/[0-9a-zA-Z]+/score,/api/player/player/[0-9a-zA-Z]+,/api/player/competition/[0-9a-zA-Z]+/ranking"
OUTFORMAT=count,method,uri,min,max,sum,avg,p99

CHANNEL:=C063N2J4W65
TMP_FILE:=tmp.txt

# セットアップ
.PHONY: init
init:
	type -p curl >/dev/null || (sudo apt update && sudo apt install curl -y)
	curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
		&& sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
		&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
		&& sudo apt update \
		&& sudo apt install gh -y
	sudo apt install -y git unzip graphviz
	wget https://github.com/tkuchiki/alp/releases/download/v1.0.10/alp_linux_amd64.zip
	unzip alp_linux_amd64.zip
	sudo install ./alp /usr/local/bin
	rm alp alp_linux_amd64.zip

# デプロイ
.PHONY: deploy1
deploy1:
	make before
	make checkout
	make build
	sudo systemctl restart nginx
	sudo systemctl restart $(SERVICE_NAME)
	sudo systemctl disable mysql
	sudo systemctl stop mysql

.PHONY: deploy2
deploy2:
	make before
	make checkout
	sudo systemctl disable nginx
	sudo systemctl stop nginx
	sudo systemctl disable $(SERVICE_NAME)
	sudo systemctl stop $(SERVICE_NAME)
	sudo systemctl restart mysql

.PHONY: deploy3
deploy3:
	make before
	make checkout
	sudo systemctl disable nginx
	sudo systemctl stop nginx
	sudo systemctl disable $(SERVICE_NAME)
	sudo systemctl stop $(SERVICE_NAME)
	sudo systemctl restart mysql

.PHONY: before
before:
	$(eval when := $(shell date "+%s"))
	mkdir -p ~/logs/$(when)
	@if [ -f $(NGX_LOG) ]; then \
		sudo mv -f $(NGX_LOG) ~/logs/$(when)/ ; \
	fi
	@if [ -f $(MYSQL_LOG) ]; then \
		sudo mv -f $(MYSQL_LOG) ~/logs/$(when)/ ; \
	fi

.PHONY: checkout
checkout:
	git chekout $(BRANCH)
	git pull origin $(BRANCH)

build:
	cd $(BUILD_DIR); \
	make isuports

.PHONY: restart
restart:
	sudo systemctl restart nginx
	sudo systemctl restart mysql
	sudo systemctl restart $(SERVICE_NAME)

# モニタリング
.PHONY: notify
notify: alp slow

## alp
.PHONY: alp
alp:
	sudo alp ltsv --file=$(NGX_LOG) --sort $(ALPSORT) --reverse -o $(OUTFORMAT) -m $(ALPM) > $(TMP_FILE)
	make slack filename=$(TMP_FILE)

## slow-query
.PHONY: slow
slow:
	sudo mysqldumpslow -s t $(MYSQL_LOG) | head -n 20 > $(TMP_FILE)
	make slack filename=$(TMP_FILE)

# DB
.PHONY: slow-on
slow-on:
	echo "set global slow_query_log_file = '$(MYSQL_LOG)'; set global long_query_time = 0; set global slow_query_log = ON;" | sudo mysql -uroot

.PHONY: slow-off
slow-off:
	echo "set global slow_query_log = OFF;" | sudo mysql -uroot

.PHONY: sql
sql:
	sudo $(MYSQL_CMD)

# Slack通知
.PHONY: slack
slack:
	curl -F file=@$(filename) -F channels=$(CHANNEL) -H "$(shell cat headers.txt)" https://slack.com/api/files.upload
