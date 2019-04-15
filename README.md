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
* Define the mask profile that which kinds of names should be masked and masked in which way by adding `MaskAttribute` to your fields/properties.
* Call APIs to mask the string
** Normaully, you can pass the request body, response body of Json or Xml/Soap directly to `MaskEngine.MaskObjectString`, which will mask everything on the fly as long as you get MaskProfile setup.

**NOTE** A global instance/singleton instance of `MaskEngine` is suggested if the Maskconfiguration is same.

### define and enable Mask Profile
Mask profile can be defined:
1. by a class, which got fields/properties with `MaskAttribute` applied with corresponding mask setting
```csharp

    public class LogKeys
    {
        [MaskAttribute(MaskFormat = "L4*8R4", FilterPattern = "\\d{16}")]
        public string AccountReference = "AccountReference";

        [MaskAttribute(MaskFormat = "*14")]
        public string Pin = "pin";

        [MaskAttribute(MaskFormat = "L4*8R4")]
        public string AccountNumber = "AccountNumber";

        [MaskAttribute(MaskFormat = "L0*6R3")]
        public string Ssn = "SSN";
        [MaskAttribute(MaskFormat = "L0*6R3")]
        public string SocialSecurityNumber = "SocialSecurityNumber";
        
        public string ClientIp = "ClientIp";
        public string ServerName = "ServerName";
        [MaskAttribute(MaskFormat = "L2*8?R0")]
        public string FirstName = "FirstName";
        [MaskAttribute(MaskFormat = "L2*8?R0")]
        public string LastName = "LastName";
        [MaskAttribute(MaskFormat = "L3*4R4")]
        public string PhoneNumber = "LastName";

        //TODO may got different formats
        [MaskAttribute(Replacement = "1970-01-01T00:00:00")]
        public string DateOfBirth = "DateOfBirth";

        [MaskAttribute(MaskFormat = "*6")]
        public string Password = "Password";
    }
```

2. by setting `MaskProfileFactory`

```csharp
            maskEngine.Configuration.MaskProfileFactory = () =>
            {
                return new Dictionary<string, MaskAttribute>
                {
                    ["mykey1"] = new MaskAttribute() { MaskFormat = "L4$4?R4" },
                    ["mykey1"] = new MaskAttribute() { MaskFormat = "L0#4?R0" },
                };
            };
            maskEngine.UseProfile<LogKeys>().FinalizeConfiguration();
```
**NOTE**: a) the profile is case-insensitive! b) after set `MaskEngineConfiguration.MaskProfileFactory`, you need to explictly call `FinalizeConfiguration()` to finalize the configuration. c) If Mask Profile got set `UseProfile<T>()` alreay and you also use `MaskEngineConfiguration.MaskProfileFactory`, duplicated mask profile item would be ignored inside `FinalizeConfiguration()`.

3. Special cases
We usually like to use key-value-pair list to passing options/parameters, and some of them may got sensitive information in it. by default, name with "key" and value with name of "value" (case in-sensitive) will be processed. For example, key with name like "AccountNumber" will be masked once your mask profile get it set.
Here, I got it enhanced that allow adding customized key-value name. Below is the example:

```csharp
            maskEngine.Configuration.KeyNameValueNameList
                .Add(new KeyValPair("customkey".ToLower(), "customvalue".ToLower()));
```

### Mask string, serialized Json or XML document
In MaskEngine it provided following APIs:
* Mask(string name, string value)
* MaskObjectString(string serializedString)
* MaskUrl(Uri uri, bool processSegemets = true)
For Url, it might be like https://tainisoft.com/username/shawn/lastname/lin?email=admin@admin.com , this case we may espect that firstname, lastname and email got masked.

BUT, actually, I think `MaskObjectString` is the most powerful method that you can benifit from it.


# Examples

# Source code
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
```csharp
static IMaskEngine MyMaskEngine;
static void Main{
        var maskEngine = new MaskEngine();
        #region custom configurations
        maskEngine.Configuration.KeyNameValueNameList
            .Add(new KeyValPair("customkey".ToLower(), "customvalue".ToLower()));
        maskEngine.Configuration.JsonStringOrXmlStringContentKeyNameChecker = (name, context) =>
        {
            return ",response_body,response_content,simple_json_str_bad,simple_xml_str_bad,simple_xml_kvp_str_bad,"
            .Contains("," + name.ToLower() + ",");
        };
        maskEngine.Configuration.MaskProfileFactory = () =>
        {
            return new Dictionary<string, MaskAttribute>
            {
                ["mykey1"] = new MaskAttribute() { MaskFormat = "L4$4?R4" },
                ["mykey1"] = new MaskAttribute() { MaskFormat = "L0#4?R0" },
            };
        };
        #endregion
        maskEngine.UseProfile<LogKeys>().FinalizeConfiguration();
        
        MyMaskEngine = maskEngine;
        
        //samples
        SimpleStringSample();
}

void SimpleStringSample(){
     var masked = MyMaskEngine("firstname", "Shawn");
     Console.WriteLine(masked);  //got "Sh***"
}
void JsonStringSample(){
     var json = "{\"firstname\":\"Shawn\",\"lastname\":\"lin\", \"options\":[{\"key\":\"ssn\",\"value\":\"123456789\"}]}";
     var masked = MyMaskEngine.MaskObjectString(json);
}

void XmlStringSample(){

}

public class LogKeys{
    //... check section # define and enable Mask Profile
}

```




