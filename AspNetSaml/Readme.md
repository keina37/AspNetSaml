# このリポジトリについて
このリポジトリは、
[https://github.com/jitbit/AspNetSaml](https://github.com/jitbit/AspNetSaml)
をフォークしたリポジトリです。

# フォークした理由と目的
Signature Reference URI=""となっているSAML Responseを返却するIdpに対応するための修正を行います。
フォーク元のライブラリではReference URIの長さが空であることを考慮しておらず、修正しないままではArgumentOutOfRangeExceptionの例外が発生します。

ただし、この修正が本来必要なものなのかはよくわかりません。
Idpとフォーク元のどちらの実装が仕様上正しいのか、確信が持てていません。（どなたか教えてくだされば幸いです）

oasisが公開している[SAMLの仕様](https://wiki.oasis-open.org/security/FrontPage)では以下のように説明されています。（2024年2月22日現在、coreの仕様書がNotFoundになっていました）
>5.4.2 References
>SAML assertions and protocol messages MUST supply a value for the ID attribute on the root element of
the assertion or protocol message being signed. The assertion’s or protocol message's root element may
or may not be the root element of the actual XML document containing the signed assertion or protocol
message (e.g., it might be contained within a SOAP envelope).
Signatures MUST contain a single <ds:Reference> containing a same-document reference to the ID
attribute value of the root element of the assertion or protocol message being signed. For example, if the
ID attribute value is "foo", then the URI attribute in the <ds:Reference> element MUST be "#foo".

（英語が苦手なので）後半部分を機械翻訳にかけると、以下の翻訳結果になりました。
>署名には、<ds:Reference>を1つ含めなければならない(MUST)。
属性値への同一文書参照を含む単一の<ds:Reference>を含まなければならない。例えば
ID 属性値が "foo" の場合、<ds:Reference> 要素の URI 属性は "#foo" でなければならない (MUST)。

Reference URI=""ではいけないようにも読めます。

しかし、W3Cの[XMLデジタル署名の仕様](https://www.w3.org/TR/xmldsig-core/)
のexample23では以下のように説明されています。
>Due to the null Reference URI in this example, the XPath transform input node-set contains all nodes in the entire parse tree starting at the root node (except the comment nodes). For each node in this node-set, the node is included in the output node-set except if the node or one of its ancestors has a tag of Signature that is in the namespace given by the replacement text for the entity &dsig;.

前半部分を機械翻訳にかけると、以下の翻訳結果になりました。
>この例では参照 URI が NULL のため、XPath 変換の入力ノードセットには、ルートノードから始まる解析ツリー全体 のすべてのノードが含まれます（コメントノードを除く）。

Reference URI=""の場合は、ルートノードから始まる解析ツリー全体の全てのノードが含まれると書かれています。（＝ルートへの参照と見なしてかまわない？）

よって、

- SAMLはXML署名を利用しているのでXML署名の仕様がベースになっていると考えて、Reference URI=""を許容する（oasisのSAML仕様書では説明を省いていて典型的なケースについてだけ記載しているのかもしれない）
- あくまでSAMLはSAML仕様書の記載に従うべきと考えて、Reference URI=""を許容しない

どちらの立場も一理あるように思えます。

Idp側の修正が望めない場合のために、ライブラリを一部修正します。

以下はフォーク元のreadmeです。

<!--![](https://github.com/jitbit/AspNetSaml/actions/workflows/dotnet.yml/badge.svg)-->

# AspNetSaml

Very short and simple SAML 2.0 "consumer" implementation in C#.

It's a *SAML client* library, not a *SAML server*. As in - allows adding SAML single-sign-on to your ASP.NET app, but *not* to provide auth services to other apps. In other words, it's a library for "service-providers" not for "identity providers".

## Installation

`Install-Package AspNetSaml`

Adds a very small .NET Standard 2.0 library (11KB dll) that works with both ASP.NET Core and the "old" ASP.NET Framework. Please refer to [releases](https://github.com/jitbit/AspNetSaml/releases) for the change log.

# Usage

## How SAML works? (please read this)

SAML workflow has 2 steps:

1. User is redirected to the SAML provider (with some magic in the query-string) where he authenticates
2. User is redirected back to your app, where you validate the payload

Here's how you do it (this example is for ASP.NET Core MVC):

## 1. Redirecting the user to the saml provider:

```c#
//this example is an ASP.NET Core MVC action method
public IActionResult Login()
{
	//TODO: specify the SAML provider url here, aka "Endpoint"
	var samlEndpoint = "http://saml-provider-that-we-use.com/login/";

	var request = new AuthRequest(
		"http://www.myapp.com", //TODO: put your app's "entity ID" here
		"http://www.myapp.com/SamlConsume" //TODO: put Assertion Consumer URL (where the provider should redirect users after authenticating)
	);

	//now send the user to the SAML provider
	return Redirect(request.GetRedirectUrl(samlEndpoint));
}
```

## 2. User has been redirected back

User is sent back to your app - you need to validate the SAML response ("assertion") that you recieved via POST.

Here's an example of how you do it in ASP.NET Core MVC

```c#
//ASP.NET Core MVC action method... But you can easily modify the code for old .NET Framework, Web-forms etc.
public async Task<IActionResult> SamlConsume()
{
	// 1. TODO: specify the certificate that your SAML provider gave you
	string samlCertificate = @"-----BEGIN CERTIFICATE-----
BLAHBLAHBLAHBLAHBLAHBLAHBLAHBLAHBLAHBLAHBLAHBLAH123543==
-----END CERTIFICATE-----";

	// 2. Let's read the data - SAML providers usually POST it into the "SAMLResponse" var
	var samlResponse = new Response(samlCertificate, Request.Form["SAMLResponse"]);

	// 3. DONE!
	if (samlResponse.IsValid()) //all good?
	{
		//WOOHOO!!! the user is logged in
		var username = samlResponse.GetNameID(); //let's get the username
		
		//the user has been authenticated
		//now call context.SignInAsync() for ASP.NET Core
		//or call FormsAuthentication.SetAuthCookie() for .NET Framework
		//or do something else, like set a cookie or something...
		
		//FOR EXAMPLE this is how you sign-in a user in ASP.NET Core 3,5,6,7
		await context.SignInAsync(new ClaimsPrincipal(
			new ClaimsIdentity(
				new[] { new Claim(ClaimTypes.Name, username) },
				CookieAuthenticationDefaults.AuthenticationScheme)));
		
		return Redirect("~/");
	}
	
	return Content("Unauthorized");
}
```

# Bonus: reading more attributes from the provider

SAML providers usually send more data with their response: username, first/last names etc. Here's how to get it:

```c#
if (samlResponse.IsValid())
{
	//WOOHOO!!! user is logged in

	//Some more optional stuff
	//let's extract username/firstname etc
	try
	{
		var username = samlResponse.GetNameID();
		var email = samlResponse.GetEmail();
		var firstname = samlResponse.GetFirstName();
		var lastname = samlResponse.GetLastName();
		
		//or read some custom-named data that you know the IdP sends
		var officeLocation = samlReponse.GetCustomAttribute("OfficeAddress");
	}
	catch (Exception ex)
	{
		//insert error handling code
		//in case some extra attributes are not present in XML, for example
		return null;
	}
}
```

# Notes about the source code

All the functionality sits in one single short file [Saml.cs](https://github.com/jitbit/AspNetSaml/blob/master/AspNetSaml/Saml.cs) other stuff in this repo are just unit tests, nuget-packaging etc. You can take that file and throw it in your project, it should work just fine.

P.S. This library has been battle-tested for years in production in our [helpdesk app](https://www.jitbit.com/helpdesk/) please check it out if you're looking for a ticketing system for your team. Cheers.
