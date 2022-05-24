---
title:  Unit testing every enum value
author: Will Andrews
date: 2019-07-15
order: 27
published: true
---

I sometimes have to unit test that a certain things happen depending on a different enum value, and there are always new enum values being added. So I don't have to change my unit tests to add in a new test for the new enum, I use a trick in xUnit to try against every enum value.

Let's say I have an enum like this.

``` c#
public enum Things
{
    Phone = 1,
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


## Extra parameters to the static method

What if I wanted to ignore some enum values, because they are irrelevant to the test. You can add parameters to the static method that returns the enum values and then pass an array of values into the MemberData attribute.

``` c#
public class EnumValueTests
{
    [Theory]
    [MemberData("enumValues", new []{ 1, 2})]
    public void TestEachType(Things thing)
    {
        Assert.IsType<Things>(thing);
    }

    public static IEnumerable<object[]> enumValues(int[] ignore)
    {
        foreach (var thing in Enum.GetValues(typeof(Things)))
        {
            if (!ignore.Contains((int)thing))
            {
                yield return new object[] { thing };
            } 
        }
    }
}
```

Now, my test results will show that only Airpods and mouse are tested. This happens because when looping through each value of my enum, I check if that numeric value is in the array of ints to ignore. It it isn't, then I return it as a value for my test to use.