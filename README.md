# Requirements

Xcode      |Swift         | Version
-----------|--------------|--------------
Xcode15    | Swift 5.9    | 1.0
<=Xcode14  | <=Swift 5.8  | [0.3.3](https://github.com/winddpan/CodableWrapper/tree/0.3.3)

# About

The project objective is to enhance the usage experience of the Codable protocol using the macro provided by Swift 5.9 and to address the shortcomings of various official versions.

# Feature

* Default value
* Basic type automatic convertible, between `String` `Bool` `Number` etc.
* Custom multiple `CodingKey`
* Nested Dictionary `CodingKey`
* Automatic compatibility between camel case and snake case
* Convenience `Codable` subclass
* Transformer

# Example

```swift
@Codable
struct BasicModel {
    var defaultVal: String = "hello world"
    var defaultVal2: String = Bool.random() ? "hello world" : ""
    let strict: String
    let noStrict: String?
    let autoConvert: Int?

    @CodingKey("hello")
    var hi: String = "there"

    @CodingNestedKey("nested.hi")
    @CodingTransformer(StringPrefixTransform("HELLO -> "))
    var codingKeySupport: String

    @CodingNestedKey("nested.b")
    var nestedB: String

    var testGetter: String {
        nestedB
    }
}

final class CodableWrapperTests: XCTestCase {
    func testBasicUsage() throws {
        let jsonStr = """
        {
            "strict": "value of strict",
            "autoConvert": "998",
            "nested": {
                "hi": "nested there",
                "b": "b value"
            }
        }
        """

        let model = try JSONDecoder().decode(BasicModel.self, from: jsonStr.data(using: .utf8)!)
        XCTAssertEqual(model.defaultVal, "hello world")
        XCTAssertEqual(model.strict, "value of strict")
        XCTAssertEqual(model.noStrict, nil)
        XCTAssertEqual(model.autoConvert, 998)
        XCTAssertEqual(model.hi, "there")
        XCTAssertEqual(model.codingKeySupport, "HELLO -> nested there")
        XCTAssertEqual(model.nestedB, "b value")

        let encoded = try JSONEncoder().encode(model)
        let dict = try JSONSerialization.jsonObject(with: encoded) as! [String: Any]
        XCTAssertEqual(model.defaultVal, dict["defaultVal"] as! String)
        XCTAssertEqual(model.strict, dict["strict"] as! String)
        XCTAssertNil(dict["noStrict"])
        XCTAssertEqual(model.autoConvert, dict["autoConvert"] as? Int)
        XCTAssertEqual(model.hi, dict["hello"] as! String)
        XCTAssertEqual("nested there", (dict["nested"] as! [String: Any])["hi"] as! String)
        XCTAssertEqual(model.nestedB, (dict["nested"] as! [String: Any])["b"] as! String)
    }
}
```

# Macro usage

## @Codable

* Auto conformance `Codable` protocol if not explicitly declared
  
  ```swift
  // both below works well
  
  @Codable
  struct BasicModel {}
  
  @Codable
  struct BasicModel: Codable {}
  ```

* Default value
  
  ```swift
  @Codable
  struct TestModel {
      let name: String
      let balance: Double = 0
  }
  
  // { "name": "jhon" }
  ```

* Basic type automatic convertible, between `String` `Bool` `Number` etc.
  
  ```swift
  @Codable
  struct TestModel {
      let autoConvert: Int?
  }
  
  // { "autoConvert": "998" }
  ```

* Automatic compatibility between camel case and snake case
  
  ```swift
  @Codable
  struct TestModel {
      let userName: String = ""
  }
  
  // { "user_name": "jhon" }
  ```

* Member Wise Init
  
  ```swift
  @Codable
  public struct TestModel {
      public let userName: String = ""
  
      // Automatic generated
      public init(userName: String = "") {
          self.userName = userName
      }
  }
  ```

## @CodingKey

* Custom `CodingKey`s
  
  ```swift
  @Codable
  struct TestModel {
      @CodingKey("u1", "u2", "u9")
      let userName: String = ""
  }
  
  // { "u9": "jhon" }
  ```

## @CodingNestedKey

* Custom `CodingKey`s in `nested dictionary`
  
  ```swift
  @Codable
  struct TestModel {
      @CodingNestedKey("data.u1", "data.u2", "data.u9")
      let userName: String = ""
  }
  
  // { "data": {"u9": "jhon"} }
  ```

## @CodableSubclass

* Automatic generate `Codable` class's subclass `init(from:)` and `encode(to:)` super calls
  
  ```swift
  @Codable
  class BaseModel {
      let userName: String
  }
  
  @CodableSubclass
  class BaseModel: BaseModel {
      let age: Int
  }
  
  // {"user_name": "jhon", "age": 22}
  ```

## @CodingTransformer

* Transformer between in `Codable` / `NonCodable` model
  
  ```swift
  struct DateWrapper {
      let timestamp: TimeInterval
  
      var date: Date {
          Date(timeIntervalSince1970: timestamp)
      }
  
      init(timestamp: TimeInterval) {
          self.timestamp = timestamp
      }
  
      static var transformer = TransformOf<DateWrapper, TimeInterval>(fromJSON: { DateWrapper(timestamp: $0 ?? 0) }, toJSON: { $0.timestamp })
  }
  
  @Codable
  struct DateModel {
      @CodingTransformer(DateWrapper.transformer)
      var time: DateWrapper? = DateWrapper(timestamp: 0)
      
      @CodingTransformer(DateWrapper.transformer)
      var time1: DateWrapper = DateWrapper(timestamp: 0)
      
      @CodingTransformer(DateWrapper.transformer)
      var time2: DateWrapper?
  }
  
  class TransformTest: XCTestCase {
      func testDateModel() throws {
          let json = """
          {"time": 12345}
          """
  
          let model = try JSONDecoder().decode(DateModel.self, from: json.data(using: .utf8)!)
          XCTAssertEqual(model.time?.timestamp, 12345)
          XCTAssertEqual(model.time?.date.description, "1970-01-01 03:25:45 +0000")
  
          let encode = try JSONEncoder().encode(model)
          let jsonObject = try JSONSerialization.jsonObject(with: encode, options: []) as! [String: Any]
          XCTAssertEqual(jsonObject["time"] as! TimeInterval, 12345)
      }
  }
  ```
