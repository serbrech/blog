---
layout: post
title: Silverlight unit testing - Test Not Initialized
date: '2011-08-21T21:15:00+02:00'
tags:
tumblr_url: http://alsagile.com/post/9217189577/silverlight-unit-testing-test-not-initialized
---
It has been a year now that I work on a silverlight project. As any respectful developer I write unit tests. But in silverlight, things are not all that easy when it comes to testing.

First of all, the unit testing framework doesn’t come out of the box. Instead, you will find it in the Silverlight Toolkit. On the bright side, this allow a separate release cycle than the one from silverlight.

Then, you might be aware that you need to run your silverlight tests in the browser, because the silverlight runtime can only run in the browser, or hosted by a wpf application since silverlight 4.
A very important part of unit testing is to have a fast feedback loop. I want to run my tests often, and therefore, I want them to run fast. Having to run them in the browser unfortunately doesn’t help it.

Those are common complains from developers who write unit tests on a silverlight project.
But the issue I will expand is different.

TestInitialize Bug

In MsTest framework, if you mark a method with a [TestInitialize] attribute, it will run before each tests. As the attribute name suggest, this is useful to initialize the state of the system before running each tests of the fixture.
Here is the problem. if the initialization fails, I really want to see my test fail. But with the silverlight testing framework, if an exception is thrown in the testinitialize method, the test will not fail. This can result in false positive, and that’s an issue because I can’t trust my tests anymore.

To work around it we have a simple base class that looks something like the following :

public class TestBase
{
    private Exception _exception;

    [TestInitialize]
    public void TestInitialize()
    {
	try
	{
	    Initialize();
	}
	catch (Exception exception)
	{
	    _exception = exception;
	}
    }

    [TestMethod]
    public void InitializeTest()
    {
	if (_exception != null)
	{
	    throw new InvalidOperationException("Failure running test setup", _exception);
	}
    }

    public virtual void Initialize() { }
}


So when I write a test fixture now, I don’t add a TestInitialize attribute above a method. Instead, I inherit from this base class and override the Initialize method. But this is obviously a dirty workaround. My tests still don’t fail. What I get is another test that tests the initialize method…
At least I have a failing test when an exception is thrown.

If you don’t like this easy workaround, you might want to have a look at the xunit.contrib project, or at nunit for silverlight. These have other advantages that I will expand in another post.
