# BytecoderWiki
**This wiki is written for the WebAssembly backend.**
**It may work with other backends but there's no guarantee.**

## Extending existing classes

Sometimes the methods or properties you want aren't there, but it's very simple to add them.

```java
public abstract class CustomCanvas extends HTMLCanvasElement {

	// The following two methods are setters and getters for the canvas.width property.

	@OpaqueProperty
	public abstract void width(float value);
	
	@OpaqueProperty
	public abstract float width();
}
```

To use your custom methods or properties simply cast a HTMLCanvasElement to CustomCanvas.

```java
((CustomCanvas) Window.window().document().getElementById("canvas")).width(1000);
```

## Implementing bridges for JavaScript objects and classes

Explanation of JavaScript data types in this wiki:
- Objects are non-instantiable (e.g. window.navigator)
- Classes are instantiable (e.g. new ArrayBuffer(...))

### Objects

```java
public abstract class Navigator implements OpaqueReferenceType {

	public static native Navigator navigator();

	@OpaqueProperty
	public abstract String userAgent();
	
	@OpaqueProperty
	public abstract boolean cookieEnabled();
	
	// If you want to have a different name in the Java code, you can
	// name the method for example beaconSend and annotate it with
	// @OpaqueMethod("sendBeacon")
	// This specific method has more types of possible arguments,
	// if you need to support those you just need to add more methods
	// but with the other `data` types
	public abstract void sendBeacon(String url, String data);
}
```

```js
// Although the import would be created because this
// class is implementing OpaqueReferenceType AND we are
// calling one of it's abstract methods in our Java code,
// not calling one would cause JavaScript errors.
// So to be 100% sure it's better to define fallback to an
// empty object.
bytecoder.imports.navigator = bytecoder.imports.navigator || {};

// This method has no arguments so it's simply called navigator
bytecoder.imports.navigator.navigator = function (thisref) {
	return bytecoder.toBytecoderReference(navigator);
};
```

```java
System.out.println(Navigator.navigator().userAgent());
```

### Classes

```java
public abstract class ArrayBuffer implements OpaqueReferenceType {

	// The @Import annotation is completely optional and
	// it removes the need for having to include the
	// parameter types in the method name on the JavaScript side.
	@Import(module = "arraybuffer", name = "create")
	public static native create(int size);
	
	@OpaqueProperty
	public int byteLength();
}
```

```js
// Read the comment on Objects in this section.
// Native methods are not abstract and if we for example
// only call ArrayBuffer.create(...) then an import would not
// be created. But if we also called ArrayBuffer.byteLength()
// it would, so it is much more secure to fallback to {}.
bytecoder.imports.arraybuffer = bytecoder.imports.arraybuffer || {};

// Passing around objects works by references, the Java code
// automatically converts this reference to an instance of
// ArrayBuffer but the JavaScript bindings have no such features.

// Warning: This method uses the @Import annotation which is why
// it is called `create` and not `createINT`.
bytecoder.imports.arraybuffer.create = function (thisref, size) {
	return bytecoder.toBytecoderReference(new ArrayBuffer(size));
};
```

```java
System.out.println(ArrayBuffer.create(6).byteLength()); // 6
```

### Taking classes as method parameters

```java
public abstract class DataView implements OpaqueReferenceType {
	public static native create(ArrayBuffer arrayBuffer);
	
	// ... opaque methods and properties ...
}
```

```js
bytecoder.imports.dataview = bytecoder.imports.dataview || {};

// create - method name, ArrayBuffer - parameter type
bytecoder.imports.dataview.createArrayBuffer = function (thisref, arraybufferref) {
	return bytecoder.toBytecoderReference(new DataView(bytecoder.toJSReference(arraybufferref)));
};
```

## Imports

Imports are methods imported from JavaScript and called from Java.

## Exports

Exports are methods exported from Java and called from JavaScript.

## Emulating classes and methods

This is done by using the @SubstitutesInClass annotation.
