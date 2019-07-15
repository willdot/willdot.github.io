---
title:  Unit testing every enum value
author: Will Andrews
date: 2019-07-15
order: 27
published: false
---

I sometimes have to unit test that a certain things happen depending on a different enum value, and there are always new enum values being added. So I don't have to change my unit tests to add in a new test for the new enum, I use a trick in xUnit to try against every enum value.

Let's say I have an enum like this.

``` c#
public enum Things
{
    Phone,
    Bottle,
    Mouse,
    Airpods
}
```

I want to be able to loop through each of the values and make sure my test works with each of them.

So in my xUnit test, I can use the "MemberData" attribute in conjunction with the "Theory" attribute. Then I can create a static method that will return each of the values of the enum and these will then be used as the theory data.

``` c#
public class ThingsUnitTests
{
    [Theory]
    [MemberData("enumValues")]
    public void TestEachType(Things thing)
    {
        Assert.IsType<Things>(thing);
    }

    public static IEnumerable<object[]> enumValues()
    {
        foreach (var thing in Enum.GetValues(typeof(Things)))
        {
            yield return new object[] { thing };
        }
    }
}
```

See how the MemberData attribute is given the name of the method that returns each of the enum values. Then the parameter to the test method is an enum value. In this test I just make sure the value is of my enum type. 
