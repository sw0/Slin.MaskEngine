# Slin.MaskEngine
This is a library developed in C# and target for .NET Framework and .NET Core. It allow developer to mask a string by given format or given profile.

# Background
In services, there are a lot calls from one to another, or 3rd party services. We need to trace the request traffic, it's commonly RESTful API calls, WCF calls. We need to get it logged into logging system, that would be helpful for aduit or troubleshooting. However, there are some sensitive information on production, we should not get it logged as plain text. So we need to mask the sensitive values. That is the main purpurse of this project that to provide a flexiable MaskEngine to mask the sensitive values for the payload in traffics.

# Introduction

## Mask Format
The built in mask format would be like `L4*6R4`, which means keep 4 of left and 4 of right part of the input string and mask the middle part with 6 `*`. 

**Examples:**

|Input String|Mask Format|Mask Result|
|:---:|-------|--------|
|12345678901234| `L4*6R4` | `1234******1234` |
|1234567890| `L4*6?R4` | `1234**7890` |
|Shawn| `L2*4R0` | `Sh****` |

Here are some exaples:
For bank account number, which is 16 digits usually, we'd like to use `L4*8R4`;
For FirstName or LastName, we'd like to use `L2*4R0`

## Usage Introduction
It's really simple that in most case you only need to:
* Initialized a global/singleton MaskEngine instance
* Define the profile that which kinds of names should be masked and masked in which way by adding `MaskAttribute` to your fields/properties.
* Call APIs to mask the string
** Normaully, you can pass the request body, response body of Json or Xml/Soap directly to MaskObjectString, which will mask everything on the fly as long as you get MaskProfile setup.


# Some code
```csharp
    public interface IMaskEngine
    {
        IMaskEngineConfiguration Configuration { get; }

        /// <summary>
        /// should be called after customized configuration got set,
        /// not been called if got custom configuration, some function may not working well.
        /// </summary>
        /// <returns></returns>
        IMaskEngine FinalizeConfiguration();

        bool IsNameInMaskList(string name);

        string Mask(string name, string value);

        string MaskObjectString(string serializedString);

        string MaskUrl(Uri uri, bool processSegements = true);
        
        /// <summary>
        /// Use a class with properties/fields got MaskAttribute setting
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <returns></returns>
        IMaskEngine UseProfile<T>() where T : class;
    }
```


Example of code:
```
```




