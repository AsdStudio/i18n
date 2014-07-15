i18n
====

Package i18n is for app Internationalization and Localization.

## Introduction

This package provides multiple-language options to users, improve user experience. Sites like [Go Walker](http://gowalker.org) and [gogs.io](http://gogs.io) use this module to implement Chinese and English user interfaces.

You can use following command to install this module:

    go get github.com/beego/i18n

## Usage

First of all, you have to import this package:

	import (
	    "github.com/Unknwon/i18n"
	)

The format of locale files is very like INI format configuration file, which is basically key-value pairs. But this module has some improvements. Every language corresponding to a locale file, for example, under `conf` folder of beego.me, there are two files called `locale_en-US.ini` and `locale_zh-CN.ini`.

The name and extensions of locale files can be anything, but we strongly recommand you to follow the style of beego.me.

## Minimal example

Here are two simplest locale file examples:

File `locale_en-US.ini`:

```
hi = hello
bye = goodbye
```

File `locale_zh-CN.ini`:

```
hi = 您好
bye = 再见
```

### Use in controller

For every request, beego uses individual goroutine to handle; therefore, you can embed a `i18n.Locale` as anonymous field to process locale operations of current request. This requires you understand the idea of `baseController` and `Prepare` method. See source file `routers/router.go` of beego.me for more details.

After accepted the request, use `Prepare` method of `baseController` to do language operations, which you only need to write same code once and use it in all the upper level controllers.

#### Register locale files

Following code is from beego.me source file `routers/init.go`:

```go
// Initialized language type list.
langs := strings.Split(models.Cfg.MustValue("lang", "types"), "|")
names := strings.Split(models.Cfg.MustValue("lang", "names"), "|")
langTypes = make([]*langType, 0, len(langs))
for i, v := range langs {
	langTypes = append(langTypes, &langType{
		Lang: v,
		Name: names[i],
	})
}

for _, lang := range langs {
	beego.Trace("Loading language: " + lang)
	if err := i18n.SetMessage(lang, "conf/"+"locale_"+lang+".ini"); err != nil {
		beego.Error("Fail to set message file: " + err.Error())
		return
	}
}
```

In this piece of code, we get languages that we want to support in the configuration file, in this case, we have `en-US` and `zh-CN`. Then we initialize a slice for users to change language option(not discuss here). Finally, we call `i18n.SetMessage` function in a loop to load all the locale files. Here you can see that why we recommand you use the name style of beego.me for locale files.

#### Initialize controller language

Following code is from beego.me source file `routers/router.go`, which decides the user languae option by order of URL specified, Cookies and browser `Accept-Language`.

```go
// setLangVer sets site language version.
func (this *baseRouter) setLangVer() bool {
	isNeedRedir := false
	hasCookie := false

	// 1. Check URL arguments.
	lang := this.Input().Get("lang")

	// 2. Get language information from cookies.
	if len(lang) == 0 {
		lang = this.Ctx.GetCookie("lang")
		hasCookie = true
	} else {
		isNeedRedir = true
	}

	// Check again in case someone modify by purpose.
	if !i18n.IsExist(lang) {
		lang = ""
		isNeedRedir = false
		hasCookie = false
	}

	// 3. Get language information from 'Accept-Language'.
	if len(lang) == 0 {
		al := this.Ctx.Request.Header.Get("Accept-Language")
		if len(al) > 4 {
			al = al[:5] // Only compare first 5 letters.
			if i18n.IsExist(al) {
				lang = al
			}
		}
	}

	// 4. Default language is English.
	if len(lang) == 0 {
		lang = "en-US"
		isNeedRedir = false
	}

	curLang := langType{
		Lang: lang,
	}

	// Save language information in cookies.
	if !hasCookie {
		this.Ctx.SetCookie("lang", curLang.Lang, 1<<31-1, "/")
	}

	restLangs := make([]*langType, 0, len(langTypes)-1)
	for _, v := range langTypes {
		if lang != v.Lang {
			restLangs = append(restLangs, v)
		} else {
			curLang.Name = v.Name
		}
	}

	// Set language properties.
	this.Lang = lang
	this.Data["Lang"] = curLang.Lang
	this.Data["CurLang"] = curLang.Name
	this.Data["RestLangs"] = restLangs

	return isNeedRedir
}
```

The variable `isNeedRedir` indicates whether user uses URL to specify the language option, for URL clean purpose, beego.me automatically set value in cookies and redirect.

The line `this.Data["Lang"] = curLang.Lang` sets user language option to template variable  `Lang` so that we can handle language in template files.

Following two lines:

	this.Data["CurLang"] = curLang.Name
	this.Data["RestLangs"] = restLangs
	
For users to change language option, see beego.me source code for more details.

#### Handle language in controller

While the `i18n.Locale` as anonymous fied to be embedded in `baseController`, we can use `this.Tr(format string, args ...interface{})` to handle language in controller.

### Handle language in template

By passing template variable `Lang` to indicate language option, you are able to do localization in template. But before that, you need to register a template function.

Following code is from beego.me source file `beeweb.go`:

	beego.AddFuncMap("i18n", i18n.Tr)
	
After that, do following with `Lang` to handle language:

	{{i18n .Lang "hi%d" 12}}
	
Code above will produce:

- English `en-US`：`hello12`
- Chinese `zh-CN`：`您好12`

## Section

For different pages, one key may map to different values. Therefore, i18n module also uses the section feature of INI format configuration to achieve section.

For example, the key name is `about`, and we want to show `About` in the home page and `About Us` in about page. Then you can do following:

Content in locale file:

```
about = About

[about]
about = About Us
```

Get `about` in home page:

	{{i18n .Lang "about"}}
	
Get `about` in about page:

	{{i18n .Lang "about.about"}}
	
### Ambiguity

Because dot `.` is sign of section in both [INI parser](https://github.com/Unknwon/goconfig) and locale files, so when your key name contains `.` will cause ambiguity. At this point, you just need to add one more `.` in front of the key.

For example, the key name is `about.`, then we can use:

	{{i18n .Lang ".about."}}
	
to get expect result.

## Helper tool

Module i18n provides a command line helper tool beei18n for simplify steps of your development. You can install it as follows:

	go get github.com/Unknwon/i18n/ui18n

### Sync locale files

Command `sync` allows you use a exist local file as the template to create or sync other locale files:

	ui18n sync srouce_file.ini other1.ini other2.ini

This commnad can operate 1 or more files in one command.

## More information

If the key does not exist, then i18n will return the key string to caller. For instance, when key name is `hi` and it does not exist in locale file, simply return `hi` as output.