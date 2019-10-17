## Yet Another Attempt to write JSON Parser

In this blog post, I have attempted to write a simple JSON parser. Better performing parsers are already available and easiest way to implement the parser is by using parser generator tools like ANTLR and only expose API's to access the parsed objects.

Nevertheless, I wanted to show how JSON parser can be written using object oriented programming concepts.

The parser takes input as the JSON document and returns "Either a value upon successful parsing or an exception indicating the failure." We have already seen the object version of "Either" data type in my  [previous blog post](https://senthilganesh.hashnode.dev/exception-handling-object-oriented-way-cjzlbrzw3000wwks1n2909li1)


```
interface Parser {
    Either<Value, JSONParserException> parse (final String document);
}

interface Value {
	default Value isString(final Consumer<String> action) { return this;}

	default Value isBool(final Consumer<Boolean> action) { return this; }

	default Value isError (final Consumer<String> action) { return this;}

	default Value isInteger (final Consumer<Integer> action) { return this; }

	default Value isDouble (final Consumer<Double> action) { return this; }

	default Value isArray (final Consumer<Value> action) { return this; }

	default Value isNull(final Thunk action) { return this; }

	default Value isJSON (final BiConsumer<Value, Value> action) { return this; }

        final static class NilValue implements Value {
	        @Override
     	        public Value isNull(final Thunk action) {
		        action.code();
		        return this;
	        }
        }
       final static class BoolValue implements Value {
	        private final boolean value;
	        BoolValue(final boolean value) { this.value = value; }
  	       @Override public Value isBool(final Consumer<Boolean> action) { 
		        action.accept(value); 
		        return this; 
	        }
        }
}
``` 
This interface looks odd. To handle different types, the usual choice is generics. But with default methods in java, I wouldn't mind implementing Value this way. The benefit of preferring default methods over generics will be evident when we look at the consumer logic later in this post.

Now we can start writing the parser implementations. 

```
final static class NilParser implements Parser {
	private final Parser other;
	NilParser (final Parser other) {
		this.other = other;
	}
	@Override
	public Either<Value, JSONParserException> parse (final String token) {
		if (token.equals("null")) {
			return Either.succ(Value.nil());
		}
		return other.parse(token);
	}
}

final static class BoolParser implements Parser {
	private final Parser other;
	BoolParser (final Parser other) {
		this.other = other;
	}
	@Override
	public Either<Value, JSONParserException> parse (final String token)  {
		if (token.equals("true") || token.equals("false")) {
			return Either.succ(Value.bool(Boolean.parseBoolean(token)));
		}
		return other.parse(token);
	}
}
final static class StringParser implements Parser {
	private final Parser other;
	StringParser(final Parser other) {
		this.other = other;
	}
	@Override
	public Either<Value, JSONParserException> parse(final String token)  {
		if (token.startsWith("\"")) {
			final int start = token.indexOf('"');
			final int end = token.lastIndexOf('"');
			if (start == 0 && end == token.length() - 1)
				return Either.succ(Value.string(token.substring(start, end + 1)));
			else {
				return Either.succ(Value.err(
						String.format("Parser error at %d\n%s\n%s\n%s",
								end + 1,
								token,
								dashes(end + 1),
								"Expecting 'EOF', '}', ':', ']', got " + token.substring(end + 1))));
			}
		}
		return other.parse(token);
	}
}
``` 
We can see that every parser implementation just concentrates on how a particular value can be parsed without any regards to other kinds of values. And each parser takes another parser in its constructor to delegate the responsibility of parsing in case if the value is not intended for current parser. This allows us to compose different parser implementations to parse multiple data types.

And the complex types such as array and object can also be defined easily.

```
final static class JSONParser implements Parser {
	private final Parser other;
	JSONParser (final Parser other) {
		this.other = other;
	}
	@Override
	public Either<Value, JSONParserException> parse(final String token)  {
		if (token.startsWith("{") && token.endsWith("}")) {
			final Map<Value, Value> map = new HashMap<>();
			final String inner = token.substring(1, token.length() - 1);
			int nbr = 0;
			int nb = 0;
			int vs = 0;
			int ks = 0;
			String key = "";
			for (int i = 0; i < inner.length(); i ++) {
				if (Character.isWhitespace(inner.charAt(i))) {
					//skip reference
				} else if (inner.charAt(i) == '{') {
					nbr ++;
				} else if(inner.charAt(i) =='}') {
					nbr --;
				} else if (inner.charAt(i) =='[') {
					nb ++;
				} else if(inner.charAt(i) ==']') {
					nb --;
				} else if (inner.charAt(i) ==',') {
					if (nbr == 0 && nb == 0) {
						ks = i + 1;
						final String _key = key;
						Parser.ALL.parse(inner.substring(vs, i).trim()).ifSuccess(value -> map.put(Value.string(_key), value));							
					}
				} else if (inner.charAt(i) ==':') {
					if (nbr == 0 && nb == 0) {
						key = inner.substring(ks, i).trim();
						vs = i + 1;
					}
				}
				if (i == inner.length() - 1) {
					if (nbr == 0 && nb == 0) {
						final String _key = key;
						Parser.ALL.parse(inner.substring(vs, i + 1).trim()).ifSuccess(value -> map.put(Value.string(_key), value));							
					}
				}
			}
			return Either.succ(Value.json(map));
		}
		return other.parse(token);
	}
``` 
We delegate the parsing responsibility to another parser whenever it detects recursive array or object structures. 

And the consumer logic is ...

```
public void testJSONValue() throws Exception { 
		
	Parser.create().parse(
			"{" +
					"name : \"Senthil\"," + 
					"Employed : true," +
					"favourites : [\"TDD\", \"OOPS\"]" + 
			"}")
	.ifSuccess(v -> v
			.isJSON((k, _v) -> _v
					.isString(__v -> System.out.printf("String Attribute (%s, %s)\n", k, __v))
					.isBool(__v -> System.out.printf("Boolean Attribute(%s, %s)\n", k, __v))
					.isArray(__v -> __v
							.isString(___v -> System.out.printf("\t String Value (%s)", ___v)))))
	.ifFailure(System.out::println);
}
``` 
We can build objects from JSON parsing as follows:

```
public void testJSONToObject() throws Exception {
	Parser.create().parse("{" + 
			"  \"name\" : \"Senthil\"," + 
			"  \"address\" : {" + 
			"    \"flatNumber\" : \"#1011\"," + 
			"    \"building\"   : \"PureOO\"," + 
			"    \"city\"       : \"Objectvile\"" + 
			"  }," + 
			"  \"cargo\"   : {" + 
			"    \"trackingID\" : \"AOL123\"," + 
			"    \"from\"       : \"Impericity\"," + 
			"    \"to\"         : \"Objectvile\"" + 
			"  }" + 
			"}")
	.ifSuccess(v -> Customer.fromJSON(v).render(System.out))
	.ifFailure(System.out::println);
}

public static Customer fromJSON (final Value node) {
	Customer.Builder bld = new Customer.Builder.CustomerBuilder();
	node
	.isJSON((k, v) -> {
		k.isString(key -> {
			if (key.equals("name")) {
				v.isString(bld::name)
			} else if (key.equals("address")) {
				v.isJSON((_k, _v) -> {
					_k
					.isString(_key -> {
						if (_key.equals("flatNumber")) {
							_v.isString(bld.address()::flatNumber);
						} else if (_key.equals("building")) {
							_v.isString(bld.address()::building);
						} else if (_key.equals("city")) {
							_v.isString(bld.address()::city);
						}
					})
				})
			} else if (key.equals("cargo")) {
				v
				.isJSON((_k, _v) -> {
					_k
					.isString(_key -> {
						if (_key.equals("trackingID")) {
							_v.isString(bld.cargo()::trackingID)
						} else if (_key.equals("from")) {
							_v.isString(bld.cargo()::from)
						} else if (_key.equals("to")) {
							_v.isString(bld.cargo()::to)
						}
					})
				})
			}
		});
	})
	.isError(System.err::println);
	return bld.build();
}
``` 

We were able to write simple JSON parser, handled the complexity of implementation with the help of object composition and also we have exposed pure object API's for JSON parsing.

Writing a generator to convert our Value object into JSON string is also simple and straightforward.

```
public interface Generator {

    String generate (final Value value);

    public static Generator create() {
        return new Simple();
    }
    
    final static class Simple implements Generator {

        @Override
        public String generate(final Value value) {
            final StringBuilder bld = new StringBuilder();
            
            value
            .isNull (() -> bld.append("null"))
            .isString (str -> bld.append("\"" + str + "\"")) 
            .isInteger (bld::append)
            .isDouble (bld::append)
            .isBool (bld::append);
            
            if (!bld.toString().isEmpty())
                return bld.toString();
            
           final List<String> array = new ArrayList<>();
            value.isArray(v -> array.add(generate(v)));
 
            if (value instanceof Value.Array) {
                bld.append("[");
                array.stream().reduce((lhs, rhs) -> lhs + "," + rhs).ifPresent(bld::append);
                bld.append("]");
                return bld.toString();
            }
           
            final Map<String, String> map = new LinkedHashMap<>();
            
            value.isJSON((k, v) -> map.put(generate(k), generate(v)));

            if (value instanceof Value.JSON) {
                bld.append("{");
                map.entrySet().stream()
                .map(e -> e.getKey() + ":" + e.getValue())
                .reduce((lhs, rhs) -> lhs + "," + rhs).ifPresent(bld::append);
                bld.append("}");
             }
            
            return bld.toString();
        }   
    }   
}

public void testArrayJSON() throws Exception {
        Assert.assertEquals(
            "[true,\"string\",[1,2]]",
            Generator.create().generate(
                Value.arr(Arrays.asList(
                    Value.bool(true),
                    Value.string("string"),
                    Value.arr(Arrays.asList(
                        Value.integer(1),
                        Value.integer(2)))))));
}
``` 


The complete implementation of this parser can be found  [here](https://github.com/senthilganeshs/jsonp) 