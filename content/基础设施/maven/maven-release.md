---
type: docs
title: "Maven Release"
linkTitle: "Maven Release"
weight: 1
---

## 1. 设置 OSSRH

### 1.1 创建账户

[Sign up for an account here](https://issues.sonatype.org/secure/Signup!default.jspa)

### 1.2 证明域名所有权

Now that you have an account the next step in the process is to prove ownership of the domain that matches the group that you’d like to publish to. Usually this is your domain name in reverse, so something like `com.company` if your domain is `company.com`. Since our developer community is at `solace.community` this meant we would publish to the `community.solace` group.

To prove that we own this domain I had to execute a few simple steps:

1. Open a [New Project Ticket](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134) with OSSRH.
2. Follow the instructions request in the ticket to addd a DNS TXT record to our domain.
3. Wait a few hours (it says it could take up to 2 business days) for the DNS TXT record to be verified.
4. Check the ticket for confirmation that domain ownership has been confirmed.
5. Make a note to comment on this ticket after your first release to enable syncing to maven central!

### 1.3 创建 Token

Now that we have permission to publish to our domain we need to create a user token for publishing. This token will be used as part of the publishing process.

To get this token do the following:

1. Login to the [OSSRH Nexus Repository Manager](https://s01.oss.sonatype.org/#welcome) w/ your OSSRH account
2. Go to your profile using the menu under your username at the top right.
3. You should see a list menu that is on the `Summary` page; change it to `User Token`. You can create your token on this page.
4. Copy & Paste this token info so you can use it later! **(Keep it private!)**

## 2. 设置 Maven

1. Ensured that my `groupId` starts with the reverse domain that we have approval to publish to! For example this is what we used, note that the `groupId` starts with `community.solace`.
2. When publishing maven projects you have releases and you have snapshots. A “release” is the final build for a version which does not change whereas a “snapshot” is a temporary build which can be replaced by another build with the same name.
3. Include a description name, description and url pointing to your repository. For example,
4. Include a license, source control info `scm`, developers and organization(I believe this is optional) information.
5. Add a profile for OSSRH which includes the `snapshotRepository` info, the `nexus-staging-maven-plugin`, and the `maven-gpg-plugin`. Note in the example below I have this profile `activeByDefault` so you don’t have to specify it when running maven commands, however you may not want to do this.
6. Include the `maven-release-plugin`, the `maven-javadoc-plugin`, the `maven-source-plugin` and the `flatten-maven-plugin` plugin.

## 3. 设置 GPG 签名

### 3.1 创建私有 Key

1. Install the gpg tool; on mac you can do this by executing the command below. If you aren’t using a mac check out the instructions [here](https://www.marcd.dev/articles/2021-03/mvncentral-publish-github)

   ```
   brew install gpg
   ```

2. Generate your key pair. You will be prompted for a “Real Name” and “Eamil Address” that you want to use with the key

   ```
   gpg --gen-key
   ```

### 3.2 共享公共 Key

1. Get your keypair identifier. To do this you need to list your keys. The key will have an identifier that looks like a random string of characters, something like *C48B6G0D63B854H7943892DF0C753FEC18D3F855*. In the command below I’ve replaced it with `MYIDENTIFIER` to show it’s location.

   ```
   MJD-MacBook-Pro.local:~$ gpg --list-keys
   /path/to/keyring/pubring.kbx
   ----------------------------------------
   pub   rsa3072 2021-03-11 [SC] [expires: 2023-03-11]
      MYIDENTIFIER
   uid           [ultimate] solacecommunity <community-github@solace.com>
   sub   rsa3072 2021-03-11 [E] [expires: 2023-03-11]
   ```

2. Distribute to a key server using the identifier found in the previous step. Note that you may want to publish to a different keyserver. The one that worked for me was hkp://keyserver.ubuntu.com:11371

   ```
   gpg --keyserver hkp://pool.sks-keyservers.net --send-keys MYIDENTIFIER
   ```

## 4. 设置 Github 密匙

基于前面的操作，在 Github 中设置以下密匙，可以以仓库或组织来设置：

- OSSRH_USERNAME：OSSRH 的登录账户名

- OSSRH_PASSWORD：OSSRH 中获得的 Token

- OSSRH_GPG_SECRET_KEY：GPG 的私有 Key，可以通过如下命令导出

  ```
  gpg --export-secret-keys -a {KEY-ID} > secret.txt
  ```

- OSSRH_GPG_SECRET_KEY_PASSWORD：GPG 私有 Key 的密码，即创建 Key 时设置的密码

## 5. 设置 Github Action

参考仓库：[infilos/maven-github-action-demo](https://github.com/infilos/maven-github-action-demo)

在该仓库中，推送 main 分支会自动触发 Maven 发布，并创建对应的 Release 日志。

## 参考连接

- https://www.marcd.dev/articles/2021-03/mvncentral-publish-github
- https://gist.github.com/sualeh/ae78dc16123899d7942bc38baba5203c
- https://github.com/chhh/sonatype-ossrh-parent/blob/master/publishing-to-maven-central.md

