Here is the translation of the provided text into English:

------

All Python programs mentioned here are written by myself. Since I only considered my own needs and didn't think about other situations, it would be better to modify them according to your own situation if you want to copy and use them.

> Code location: https://github.com/Leah-98/Python-Automation/tree/main/Gitbook

The biggest advantage of processing your notes into Gitbook is the ability to search globally.

# Step 1: Prepare the MD Note Repository

**1. Convert all files to MD or reference them in MD files**

Gitbook can only render Markdown pages.

Some of my old notes were exported from Evernote and Youdao Cloud, which were in web format. I converted the web format to PDF and referenced it in the README.md folder (append_other_files_to_readme.py).

There are also some scattered TXT notes, which I directly changed the suffix to MD (txt_to_md.py).

**2. Place the images referenced in MD in a subfolder at the same level as the MD file, and name the subfolder after the MD file with "_imgs" appended**

The image references in the previous notes were messy. Some were web images, and some were absolute paths written by Typora. When operating on the same note repository on different computers, the absolute paths were inconsistent. Some were placed in the "imgs" folder with relative paths, but the "imgs" folder was located in various positions within the note repository, making it messy.

I used a Python program to unify and place them in a subfolder at the same level as the MD file, naming the subfolder after the MD file with "_imgs" appended, and rewriting the image references in MD to relative paths (reformat_imgs.py).

Also, many of my book excerpts are directly taken from physical books. The phone photos are too large and need to be compressed (compress_pictures.py).

**3. Generate SUMMARY.md**

Arrange all MD files and write them in SUMMARY.md. If it's a subfolder and doesn't have README.md, create one yourself (generate_summary.py).

This is better than gitbook init, which has various bugs and cannot generate a summary based on all MD files.

# Step 2: Prepare the Gitbook Setup Environment

I followed some online tutorials and encountered many issues.

> This tutorial is too simple. You can take a look to get a general idea, but don't follow its procedures:
>
> https://tonydeng.github.io/gitbook-zh/
>
> This one is more detailed and better:
>
> https://1927344728.github.io/fed-knowledge/tools/Gitbook/

**0. Preparing for pitfalls**

At first, I directly downloaded and installed from the official website:

> Official website: https://nodejs.org/en

However, the version from the official website (v18.x) is too new, causing various adaptation issues with gitbook. Later, I downgraded to v.10.x to use gitbook normally.

I encountered an error when installing gitbook-cli after installing v18.x version of nodejs and calling gitbook init in the command line:

```
shellCopy code/home/travis/.nvm/versions/node/v12.18.3/lib/node_modules/gitbook-cli/node_modules/npm/node_modules/graceful-fs/polyfills.js:287
      if (cb) cb.apply(this, arguments)
                 ^
TypeError: cb.apply is not a function
    at /home/travis/.nvm/versions/node/v12.18.3/lib/node_modules/gitbook-cli/node_modules/npm/node_modules/graceful-fs/polyfills.js:287:18
    at FSReqCallback.oncomplete (fs.js:169:5)
```

> Solution: https://stackoverflow.com/questions/64211386/gitbook-cli-install-error-typeerror-cb-apply-is-not-a-function-inside-graceful

You need to update the graceful-fs version.

When I tried to update with npm, I encountered another issue where any command would result in an error "ERR! Cannot read property 'insert' of undefined".

> Solution: https://github.com/npm/cli/issues/4876

You need to change the mirror to [https://registry.npmmirror.com](https://registry.npmmirror.com/). If you also need to set a proxy, there is a tutorial here:

> Setting npm proxy: https://blog.csdn.net/yanzi1225627/article/details/80247758

After updating, calling any gitbook command resulted in no response, no error, and no information. The local directory remained unchanged. I then downgraded nodejs to 10.x.

During this process, there was also an issue that gitbook and gitbook-cli could not coexist. You need to uninstall gitbook to use gitbook-cli (the command-line tool for gitbook). After installing gitbook-cli, the first time you call the gitbook command, like checking the version with `gitbook -V`, gitbook-cli will install the corresponding version of gitbook.

Since I got stuck at the "install gitbook" step every time I called the gitbook command and checked online for a solution, I found that you can clone gitbook's github repository and then install and link it yourself:

```
shellCopy codegit clone https://github.com/GitbookIO/gitbook.git
cd gitbook
npm install
npm link
```

Firstly, the instructions in the gitbook repository say to install with `bun` instead of `npm`. Secondly, this installation is equivalent to `npm install gitbook`, which conflicts with gitbook-cli. Therefore, this method is incorrect.

Here is the correct method:

**1. Install nvm (Node.js version manager) and Node.js v10.x**

Choose one based on your system:

```
shellCopy codecurl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

bash
Copy code
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```

Install Node.js v10.x and use it:

```
shellCopy codenvm install 10
nvm use 10
```

I also encountered an issue here. In the above steps, the npm version compatible with Node.js v10.x will be installed. However, I previously used v18.x. After switching to Node.js v10.x, the npm version did not change. Any npm command would result in an error. Later, I switched back to v18, uninstalled npm, which uninstalls the latest version of npm, and then switched to v10.x, allowing me to use the older version of npm.

```
shellCopy codenvm use 18
npm uninstall npm -g
```

The `-g` means the operation is carried out globally, which will be used in the following commands.

**2. Install and use gitbook-cli & gitbook**

```
shellCopy code
npm install gitbook-cli -g
```

The `npm install -g` command is used to globally install Node.js modules. This means the installed modules will be placed in a specific directory, usually the global `node_modules` directory of your system, not in the `node_modules` directory of the current project.

If you don't use the `-g` (global) parameter in the `npm install` command, the command will install the specified package in the `node_modules` folder of the current project, not in the global environment.

```
shellCopy code
gitbook -V
```

Check the gitbook version. The first time you call any gitbook command, it will attempt to install gitbook.

Running `gitbook -V` again will show the versions of gitbook-cli and gitbook.

Note: Do not install gitbook with `npm install gitbook` directly; use gitbook-cli instead.

**3. Preview the Gitbook website and make adjustments**

Switch the command line to the root directory of your MD source files and run the following command to automatically generate README.md and SUMMARY.md:

```
shellCopy code
gitbook init
```

Note: Do not include the character "#" in the directory names and filenames under the MD files, meaning the generated SUMMARY.md should not contain the "#" character. If it does, it will cause the links in the gitbook page's left navigation bar to not work.

Also, do not have consecutive curly braces in the MD files, like {{ and }}.

> Related instructions: [https://wkcom.gitee.io/gitbook%E7%9A%84%E4%BD%BF%E7%94%A8/3%E3%80%81gitbook%E8%BF%9E%E7%BB%AD%E5%A4%A7%E6%8B%AC%E5%8F%B7%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E5%BC%8F.html](https://wkcom.gitee.io/gitbook的使用/3、gitbook连续大括号的解决方式.html)

Adding a space between the curly braces will resolve the issue.

Preview the gitbook website:

```
shellCopy code
gitbook server
```

Open the local port in the browser to view the page:

```
shellCopy code
http://localhost:4000/
```

# Step 3: Set up .io online website

There are many solutions; I chose to directly upload the static website to Github.

Generate the gitbook static website, run the following command in the root directory of your MD source files:

```
shellCopy code
gitbook build
```

This will generate a `_book` directory, containing the static website code.

As long as the repository has a `gh-pages` branch, you can access Gitbook Pages with the following link:

```
shellCopy code
https://githubusername.github.io/reponame
```

So the steps are:

1. Create an empty Github repository.
2. Check out a `gh-pages` branch, upload the static website.
3. I need multiple gitbooks, so I created a subfolder in the root directory and uploaded the corresponding static website in this subfolder.

Then, I can access multiple Gitbooks with the following paths:

```
shellCopy codehttps://githubusername.github.io/reponame/subfoldername1
https://githubusername.github.io/reponame/subfoldername2
```

