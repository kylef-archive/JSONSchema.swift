# JSON Schema

An implementation of [JSON Schema](http://json-schema.org/) in Swift.
Supporting JSON Schema Draft 4, 6, 7, 2019-09, 2020-12.

The JSON Schema 2019-09 and 2020-12 support are incomplete and have gaps with
some of the newer keywords.

JSONSchema.swift does not support remote referencing [#9](https://github.com/kylef/JSONSchema.swift/issues/9).

## Installation

JSONSchema can be installed via [CocoaPods](http://cocoapods.org/).

```ruby
pod 'JSONSchema'
```

## Usage
### Basic

```swift
import JSONSchema

try JSONSchema.validate(["name": "Eggs", "price": 34.99], schema: [
  "type": "object",
  "properties": [
    "name": ["type": "string"],
    "price": ["type": "number"],
  ],
  "required": ["name"],
])
```
### JSONSchemaValidatorHelper.swift example
Inside the `Example` folder you can find `JSONSchemaValidatorHelper.swift`.
It shows how to load a json schema from a url and then validate it agains an object that confirms to the [Codable](https://developer.apple.com/documentation/swift/codable) protocol.

```swift
struct JSONSchemaValidatorHelper {
    
    private var fetchSchemaCancellable : AnyCancellable?
    
    ///loads the json schema from the specified url (using combine framework aka publishers)
    mutating func loadJsonSchema(url: URL, completion: @escaping ([String: Any]?) -> Void) {
        fetchSchemaCancellable = URLSession.shared.dataTaskPublisher(for: url)
            //after receiving the data from the load, convert it to jsonObject that is representing our schema
            .map { data, urlResponse in
                try? JSONSerialization.jsonObject(with: data, options: [])
            }
            //don't care about the url loading errors, just return nil schema if this happens
            .replaceError(with: nil)
            //go and call ourpassed in completion handler when finished
            .sink { value in
                completion(value as? [String:Any])
            }
    }
    
    ///converts the specified encodable object to json and validates it agains the specified schema url
    mutating func validate<Object>(encodableObject: Object, againstSchemaUrl schemaUrl: URL, completion: @escaping (ValidationResult?) -> Void ) where  Object: Encodable {
        //first load our schema
        loadJsonSchema(url: schemaUrl) { schema in
            //check schema loaded successfully
            guard let schema = schema else {
                completion(ValidationResult.invalid(["Could not schema"]))
                return
            }
            //convert our object to json
            guard let jsonData   = (try? JSONEncoder().encode(encodableObject)),
                  let jsonEquivalentFoundationObjects = (try? JSONSerialization.jsonObject(with: jsonData, options: [])) else {
                completion(ValidationResult.invalid(["Could not convert 'encodableObject' to json equivalent foundation objects"]))
                return
            }
            //perform validation and let our completion handler know the result
            completion(JSONSchema.validate(jsonEquivalentFoundationObjects, schema: schema))
        }
    }
    
}
```

This is how to use it:

```swift
        let schemaUrl = URL(string: "https://path.to.your.schema.json")!
        jsonSchema.validate(encodableObject: snookerTable, againstSchemaUrl: schemaUrl) { result in
            print(result)
        }
```

## Error handling

Validate returns an enumeration `ValidationResult` which contains all
validation errors.

```python
print(try validate(["price": 34.99], schema: ["required": ["name"]]).errors)
>>> "Required property 'name' is missing."
```

## License

JSONSchema is licensed under the BSD license. See [LICENSE](LICENSE) for more
info.

