---
layout: post
identifier: fbabdd69b6534916aa86367a0ac3cbce
title: 'Stop the war between NUnit and MSTest : Make them friends'
date: '2010-03-09T00:00:00+01:00'
tags:
tumblr_url: http://alsagile.com/post/9705266525/make-nunit-and-mstest-play-together
---
Note : this is an old article from my previous blog that I ported here

Recently, Roy Osherove blogged NUnit vs. MsTest: NUnit wins for Unit Testing in an answer to this StackOverflow question  My team is using TFS2008 (and soon 2010) for source control. When I introduced Continuous Integration and Unit Testing in my team, I considered using NUnit and a separate continuous integration server like CruiseControl.NET.

Setting this up is a burden. I quickly changed my mind and decided to stick with TFS for the continuous integration. When it comes to the testing framework, you can get TFS to run NUnit, but you will loose the the nice reports including detailed tests results, code coverage data, and so on… no point.


So I run my test using MSTest. Just like many other, and as Roy points out, the API from Assert API from NUnit is much nicer than MSTest one.


Roy Osherove :
MsTest’s ExpectedException attribute has a bug where the expected message is never really asserted even if it’s wrong - the test will pass.
Nunit has an Assert.Throws API to allow testing an exception on a specific line of code instead of the whole method (you can easily implement this one yourself though)
Nunit contains a fluent version of Assert api (as already mentioned - Assert.That..)
…NUnit was created SOLELY for the idea of unit testing. MsTest was created for Testing - and also a bit of unit testing.



Here is the trick, it’s really so easy. I wonder how come nobody mentions it anywhere. so hit me if that’s a No-No, but I haven’t got any problem so far (most recent MVC project having about 150 Unit Tests and growing):




using Microsoft.VisualStudio.TestTools.UnitTesting;
using Assert = NUnit.Framework.Assert;
…
    [TestClass]   //MSTest
    public class AlsagileTests
    {
	[TestMethod]   //MSTest
	public void MakeFriends_NUnitAndMSTestWorkTogetherFTW_ReturnsTrue()
	{
	    var blog= new Alsagile();
	    bool areFriends = blog.MakeFriends();
	    Assert.That(areFriends, Is.True); //NUnit
	}


	[TestMethod]   //MSTest
	public void Leave_ReaderLeavesMyBlog_ThrowsActionNotSupportedException()
	{
	    var blog = new Alsagile();
	    Assert.That(() => blog.Leave(),
	    Throws.TypeOf(typeof(ActionNotSupportedException)).
	    With.Message
		   .EqualTo("You are not following me on Twitter yet!"));
	}
}



That’s it! The test is now using MSTest test runner, but all the assert will be performed by NUnit. I can take full advantage of the great Fluent Assert.That(…) API and other Assert methods from Nunit, and still use the automated build and tests and report power from TFS.

Follow me on twitter to avoid the exception : @serbrech
