---
title: 基于gitlab的webhook搭建的commit检查服务
tags: [git,杂项,应用]
categories: [git]
date: 2019-03-03 17:53:14
description: 前言、流程、相关代码
---

# 前言

最近在项目里遇到了一点问题：某个里程碑因为git上merge失误的关系，不小心把下个里程碑的代码跟混进去了。由于缺乏有效的检查，加上QA童鞋回归时并没有测试某些极端的情况，在线上环境出了点问题，最后不得不重新发版解决。

痛定思痛，博主这边打算找些自动化的检查机制来避免以后再出现这种问题。
我们这边版本管理工具用的是Redmine，其实我们组里一直都有在commit信息上带上相关单号的好习惯（如：[#000000]xxxxxxx），不过一般只有进行codeReview的时候才会用到
考虑到最近项目迁移到了gitlab上，在某些特殊的时间能够触发webhook调用一些特定的url，博主决定自己搭一个nodejs服务器，在有push至gitlab的时候，对其commit的信息进行检查

# 流程

## 获取commit信息

当有push推送至gitlab服务器时，会触发gitlab的webhook中的Push events，具体的官方doc如下：
https://docs.gitlab.com/ee/user/project/integrations/webhooks.html#push-events

根据doc可以看到，请求的body中会通过json格式带上一些我们感兴趣的信息
值得注意的是，commits字段虽然有commit相关的信息，但是由于性能的缘故，只会有前20条相关的记录，而total_commits_count字段则会提供完整的commit数量

这该如何是好，总不能说当我们一次push太多commit去服务器的时候校验就失效了吧
考虑到像develop与master分支一般都会有上百次commit一口气push上来，这个问题还是不容忽视的
在参考了Jenkins构建项目时，其实是clone了一份代码至本地仓库来进行操作之后，博主也决定效仿：
决定在nodejs服务上也clone一份本地仓库，当发生push时通过pull指令更新，最终使用log指令来获取完整的commit信息

值得注意的是，这里为了获取正确的git log顺序，使用了--topo-order拓扑排序的参数（这里主要为了解决merge之后git log直接拉取数据的话，前几条并不一定是最新在当前分支上的修改的问题，说实话参数的doc介绍没看的太懂）

## 流程设计

流程图如下：
![流程图](1.png)

这里把实际检查工作放到一个队列中依次执行的原因主要是：
1. 检查工作中包含多项异步处理（如：读取文件、更新代码、请求Redmine api数据）会比较耗时，此处应该尽快回复gitlab的webhook，避免webhook以为调用失败再次请求
2. 由于使用本地仓库获取log日志的缘故，多个检查工作同时执行相关git指令可能会导致本地仓库出问题

# 相关代码

## 仓库文件编辑工具

```js
const fs = require("fs");

const CONFIG_FILE_SUFFIX = "_config";

exports.BRANCHES = "branches";
exports.VERSION = "version";

/**
 * 检查仓库的配置文件是否存在
 * @param repositoryName 仓库名
 * @param callback 回调是否存在文件
 */
exports.hasRepositoryConfigFile = function (repositoryName, callback) {
    fs.exists(getFileName(repositoryName), function (exists) {
        callback(exists);
    });
};

/**
 * 读取配置文件
 * @param repositoryName 仓库名
 * @param callback 回调解析的json对象
 */
exports.loadConfigFileData = function (repositoryName, callback) {
    exports.hasRepositoryConfigFile(repositoryName, function (exists) {
        if (exists) {
            fs.readFile(getFileName(repositoryName), 'utf8', function (err, data) {
                let object = {};
                if (err) {
                    callback(object);
                    return;
                }
                try {
                    object = JSON.parse(data);
                } catch (e) {
                    // ignore
                }
                callback(object);
            })
        } else {
            callback({});
        }
    });
};

/**
 * 写入配置文件
 * @param repositoryName 仓库名
 * @param object 配置对象
 * @param callback 回调写入的错误
 */
exports.writeConfigFileData = function (repositoryName, object, callback) {
    fs.writeFile(getFileName(repositoryName), JSON.stringify(object), err => {
        callback(err);
    })
};

function getFileName(repositoryName) {
    return "./" + repositoryName + CONFIG_FILE_SUFFIX;
}
```

## webhook校验代码

```js
const http = require("http");
const querystring = require("querystring");
const fs = require("fs");
const exec = require("child_process").exec;
const configFileUtils = require("./configFileUtils");
const EventEmitter = require("events").EventEmitter;
const logUtils = require("./logUtils");

const TOKEN_KEY = "x-gitlab-token";
const TOKEN = "Hello, Thank you, Thank you very much!";
const CHECK_EVENT = "executeShell";
const INVALID_ORDER_KEY = "invalid_order";

const orderPattern = new RegExp("\\[#\\d+]");
const event = new EventEmitter();
const dataList = [];

// 网络请求处理
exports.handleRequest = function (req, res) {
    let data = "";
    req.on("data", function (chunk) {
        data += chunk;
    });
    req.on("end", function () {
        const tokenValid = req.headers[TOKEN_KEY] === TOKEN;

        try {
            logUtils.log("### get new data from git hook ! ###");
            const body = JSON.parse(data);
            if (tokenValid && body["object_kind"] === "push") {
                const repositoryName = body["repository"]["name"];
                configFileUtils.hasRepositoryConfigFile(repositoryName, function (exist) {
                    if (exist) {
                        dataList.push(body);
                        event.emit(CHECK_EVENT);
                    } else {
                        logUtils.error(`the repository ${repositoryName} can not get config file, just ignore it`);
                    }
                });
            } else {
                logUtils.error("token invalid or not push event, just ignore it");
            }
        } catch (e) {
            // ignore
            logUtils.error("catch error in parse data ? , just ignore it");
        }

        res.writeHead(tokenValid ? 200 : 404);
        res.end();
    });
    req.on("error", function (err) {
        logUtils.error(`req on error : ${err}`);
    });
};

// 开始检查提交内容
let checking = false;

event.on(CHECK_EVENT, function () {
    if (checking || dataList.length === 0) {
        logUtils.log("### is in checking or not more check to do, pass check event ! ###");
        return;
    }
    checking = true;
    logUtils.log("begin check");

    try {
        const data = dataList.pop();

        const repositoryName = data["repository"]["name"];
        const repositoryUrl = data["repository"]["url"];
        const branchName = data["ref"].replace("refs/heads/", "");
        const hashCode = data["after"];
        const commitCount = data["total_commits_count"];
        const userName = data["user_name"];
        const userEmail = data["user_email"];
        let configObject;

        logUtils.log(`hook data repositoryName(${repositoryName}) repositoryUrl(${repositoryUrl}) branchName(${branchName})`);
        logUtils.log(`hook data commitCount(${commitCount}) userName(${userName}) userEmail(${userEmail}) hashCode(${hashCode})`);

        // 加载配置文件
        loadConfigFile(repositoryName).then(object => {
            configObject = object;
            return checkBranchName(branchName, configObject); // 检查分支名，是否要检测的分支
        }).then(() =>
            updateLocalRepository(repositoryName, repositoryUrl) // 更新本地仓库
        ).then(() =>
            checkOutBranch(repositoryName, branchName, hashCode) // 分支切换
        ).then(() =>
            collectCommitMessages(repositoryName, commitCount, configObject) // 收集提交信息，返回 orderMessagesMap 与 configObject
        ).then(orderMessagesMap =>
            checkCommitMessagesByRedmine(repositoryName, orderMessagesMap, configObject) // 检查提交信息
        ).then(badOrderMessagesAndVersionMap => {
            notifyUserBadCommitMessages(badOrderMessagesAndVersionMap, userName, repositoryName, branchName, userEmail, configObject); // 错误提交信息通知
            endCheck();
        }).catch(function (message) {
            logUtils.error(message); // 异常打印
            endCheck();
        });
    } catch (e) {
        // ignore
        logUtils.error(e);
        endCheck();
    }
});

function endCheck() {
    if (checking) {
        logUtils.log("end check");
        checking = false;
        event.emit(CHECK_EVENT);
    }
}

function loadConfigFile(repositoryName) {
    return new Promise((resolve, reject) => {
        configFileUtils.loadConfigFileData(repositoryName, object => {
            if (Object.keys(object).length > 0) {
                logUtils.log("load config file success");
                resolve(object);
            } else {
                reject(`empty config file for ${repositoryName}`);
            }
        })
    })
}

function checkBranchName(branchName, configObject) {
    return new Promise((resolve, reject) => {
        try {
            const branches = configObject[configFileUtils.BRANCHES];
            for (let i = 0; i < branches.length; i++) {
                if (branches[i] === branchName) {
                    logUtils.log("check branch name success");
                    resolve();
                    return;
                }
            }
        } catch (e) {

        }
        reject("not valid branch !")
    })
}

function updateLocalRepository(repositoryName, repositoryUrl) {
    return new Promise((resolve, reject) => {
        fs.exists("./" + repositoryName, exists => {
            const cmd = exists ? "git pull" : `git clone ${repositoryUrl}`;
            const cwd = exists ? `./${repositoryName}/` : ".";
            exec(cmd, {cwd: cwd}, error => {
                if (error) {
                    logUtils.error(error);
                    reject("error in update local repository !");
                } else {
                    logUtils.log("update local repository success");
                    resolve();
                }
            });
        })
    });
}

function checkOutBranch(repositoryName, branchName, hashCode) {
    return new Promise((resolve, reject) => {
        const cwd = `./${repositoryName}/`;
        exec("git checkout -B check remotes/origin/" + branchName, {cwd: cwd}, error => {
            if (error) {
                logUtils.error(error);
                reject("error in check out branch !");
            } else {
                // 检查当前位置的哈希值是否正确
                exec("git log --pretty=format:\"%H\" -n 1", {cwd: cwd}, (error, stdout) => {
                    if (error) {
                        logUtils.error(error);
                        reject("error in check out branch !");
                    } else if (stdout === hashCode) {
                        logUtils.log("check out branch success !");
                        resolve();
                    } else {
                        reject(`the sha hash in current commit (${stdout}) is not match !`);
                    }
                });
            }
        });
    });
}

function collectCommitMessages(repositoryName, commitCount) {
    return new Promise((resolve, reject) => {
        const cwd = "./" + repositoryName + "/";
        exec(`git log --pretty=format:\"%s\" --topo-order -n ${commitCount}`, {cwd: cwd}, (error, stdout) => {
            if (error) {
                logUtils.error(error);
                reject("error in collect commit messages !");
                return;
            }
            const messages = stdout.split("\n");
            const orderMessagesMap = {};
            for (let i = 0; i < messages.length; i++) {
                logUtils.log(`collect message index ${i} : ${messages[i]}`);
                // Merge操作，直接跳过
                if (messages[i].startsWith("Merge")) {
                    continue;
                }
                // 解析单号
                const resultArray = orderPattern.exec(messages[i]);
                if (!resultArray) {
                    if (!orderMessagesMap[INVALID_ORDER_KEY]) {
                        orderMessagesMap[INVALID_ORDER_KEY] = [];
                    }
                    orderMessagesMap[INVALID_ORDER_KEY].push(messages[i]);
                    continue;
                }
                // 单号解析成功
                const order = resultArray[0].substr(2, resultArray[0].length - 3);
                if (!orderMessagesMap[order]) {
                    orderMessagesMap[order] = [];
                }
                orderMessagesMap[order].push(messages[i]);
            }
            logUtils.log("collect commit messages success !");
            resolve(orderMessagesMap);
        });
    });
}

// 这个key可以去redmine的/my/account页面右边获取
const REDMINE_KEY = "xxx";

function checkCommitMessagesByRedmine(repositoryName, orderMessagesMap, configObject) {
    return new Promise((resolve, reject) => {
        const orders = [];
        const badOrderMessagesAndVersionMap = {}; // order -> messages , version
        for (let order in orderMessagesMap) {
            if (!orderMessagesMap.hasOwnProperty(order)) {
                continue;
            }
            if (order === INVALID_ORDER_KEY) {
                badOrderMessagesAndVersionMap[order] = {messages: orderMessagesMap[order], version: ""};
            } else if (order !== "0") { // 过滤单号为 0 的提交
                orders.push(order);
            }
        }
        let ordersStr = "";
        for (let i = 0; i < orders.length; i++) {
            if (i !== 0) {
                ordersStr += "|";
            }
            ordersStr += orders[i];
        }
        if (ordersStr.length === 0) { // 没有需要检查的单号
            logUtils.log("check commit messages by redmine success !");
            resolve(badOrderMessagesAndVersionMap);
        } else {
            // 请求redmine获取单号的里程碑
            const data = {
                key: REDMINE_KEY,
                issues_id: ordersStr,
                status_id: "*",
                limit: orders.length
            };
            const option = {
                hostname: "xxx",
                port: xxx,
                path: "/issues.json?" + querystring.stringify(data),
                method: "GET"
            };
            http.request(option, res => {
                logUtils.log(`redmine status code : ${res.statusCode}`);
                let data = "";
                res.on("data", chunk => {
                    data += chunk;
                });
                res.on("end", () => {
                    try {
                        const object = JSON.parse(data);
                        const orderVersionMap = {};
                        if (object["issues"]) {
                            for (let i = 0; i < object["issues"].length; i++) {
                                const issue = object["issues"][i];
                                orderVersionMap[issue["id"]] = issue["fixed_version"]["name"].trim();
                            }
                        }
                        // 检查每个单号的里程碑是否合法
                        const validVersion = configObject[configFileUtils.VERSION].trim();
                        for (let i = 0; i < orders.length; i++) {
                            const order = orders[i];
                            // redmine 解析里程碑失败
                            if (!orderVersionMap[order]) {
                                logUtils.log(`check commit messages order ${order} fail with no version !`);
                                badOrderMessagesAndVersionMap[order] = {
                                    messages: orderMessagesMap[order],
                                    version: ""
                                };
                                continue;
                            }
                            const version = orderVersionMap[order];
                            if (version !== validVersion) {
                                logUtils.log(`check commit messages order ${order} fail with version : ${version} !`);
                                badOrderMessagesAndVersionMap[order] = {
                                    messages: orderMessagesMap[order],
                                    version: version
                                };
                            }
                        }
                        logUtils.log("check commit messages by redmine success !");
                        resolve(badOrderMessagesAndVersionMap);
                    } catch (e) {
                        logUtils.error(e);
                        reject("check commit messages by redmine fail !")
                    }
                })
            }).end();
        }
    });
}

function notifyUserBadCommitMessages(badOrderMessagesAndVersionMap, userName, repositoryName, branchName, userEmail, configObject) {
    // 解析提交信息，生成错误信息
    let badCommitsMessages = [];
    let errorMessages = [];
    let counter = 0;
    for (let order in badOrderMessagesAndVersionMap) {
        if (!badOrderMessagesAndVersionMap.hasOwnProperty(order)) {
            continue;
        }
        counter++;
        const messages = badOrderMessagesAndVersionMap[order]["messages"];
        for (let i = 0; i < messages.length; i++) {
            badCommitsMessages.push(messages[i]);
        }
        if (order === INVALID_ORDER_KEY) {
            errorMessages.push("存在不规范的提交信息编写");
        } else if (badOrderMessagesAndVersionMap[order]["version"]) {
            errorMessages.push(`单号 ${order} 的周里程碑为 ${badOrderMessagesAndVersionMap[order]["version"]}`);
        } else {
            errorMessages.push(`单号 ${order} 无法从redmine解析出正确的周里程碑`);
        }
    }
    // 无需提醒，直接退出
    if (counter === 0) {
        logUtils.log("no need to notify, just return !");
        return;
    }
    // 拼接字符串
    let errorMessagesStr = "";
    for (let i = 0; i < errorMessages.length; i++) {
        errorMessagesStr += `${i + 1}. ${errorMessages[i]}\n`;
    }
    let badCommitsMessagesStr = "";
    for (let i = 0; i < badCommitsMessages.length; i++) {
        badCommitsMessagesStr += `${i + 1}. ${badCommitsMessages[i]}\n`;
    }
    let notifyMessage = `警告！本次合法的周里程碑为 "${configObject[configFileUtils.VERSION].trim()}"\n${userName}童鞋，您刚刚往 ${repositoryName} 仓库中的 ${branchName} 分支推送的代码内，包含以下问题：\n${errorMessagesStr}\n请对以下提交信息进行确认：\n${badCommitsMessagesStr}`;
    // 使用popo机器人进行通知
    const uid = userEmail.indexOf("@") !== -1 ? userEmail.split("@")[0] : null;
    if (!uid) {
        logUtils.log("can not notify with empty uid, just return !");
        return;
    }
    const data = {
        uid: uid,
        msg: notifyMessage
    };
    const option = {
        hostname: "xxx",
        port: xxx,
        path: "/api/post_popo?" + querystring.stringify(data),
        method: "GET"
    };
    http.request(option).end();
    logUtils.log("send notify success !");
}
```