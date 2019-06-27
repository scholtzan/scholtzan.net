+++
title = "Running Python Code in BigQuery"
date = "2019-06-26"
type = "post"
tags = ["BigQuery", "Python", "Mozilla", "Open Source"]
+++

BigQuery supports user-defined functions written in JavaScript and by using WebAssembly it is [even possible to run C code](https://blog.sourced.tech/post/calling-c-functions-from-bigquery/). While having support for C is quite neat a more widely used language when it comes to data analysis and processing is Python with its scientific library ecosystem making the lifes of data scientists much easier. So, wouldn't it be cool to bring Python into BigQuery? That was the motivation behind my internship project at Mozilla.

## Pyodide and BigQuery Limitations

Starting out with this project, the initial idea was to use [Pyodide](https://github.com/iodide-project/pyodide) which is the Python stack compiled to WebAssembly including various scientific libraries, such as NumPy or Pandas. Pyodide runs in the web browser and is used as part of [Iodide](https://alpha.iodide.io/), a tool to write interactive documents using web technologies. However, after some testing and playing around it became clear that BigQuery has various limitations related to UDFs that make using Pyodide not possible:

* The maximum inline code blob size is 32 KB.
* External code files can only have a size of up to 1 MB.
* The total size of external resources is max. 5 MB.
* UDFs timeout after 60s.
* Only about 40 MB of memory is available for JavaScript processing environment.
* The JavaScript environment does not support `async` in Standard SQL and does not support certain other functions for loading external resources, such as `require` or `fetch`.


## MicroPython

After spending some time looking into alternatives to run Python in WebAssembly or transforming Python code to JavaScript, I came across [MicroPython](https://github.com/micropython/micropython). MicroPython is optimized to run on embedded systems and even offers a [port to JavaScript](https://github.com/micropython/micropython/tree/master/ports/javascript) using [Emscripten](https://emscripten.org/). MicroPython, however, has some limitations such as no Python Standard Library and no support for importing packages that have not been adapted for MicroPython. It is less powerful than the standard Python but also a lot smaller and less memory hungry.

MicroPython did not run out of the box in BigQuery. I had to make some changes to the MicroPython code since it used `async` to initialize WebAssembly, accessed DOM elements which are not available in BigQuery and tried to load external libraries which I instead embedded into the code. WebAssembly can be initialized without `async` by doing the following:

```js
const importObject = { env };
const bytes = new Uint8Array([...]);    // byte representation of wasm 
var myModule = new WebAssembly.Module(bytes);
var myInstance = new WebAssembly.Instance(myModule, importObject);
``` 

The JavaScript port of MicroPython results in `micropython.js`, which contains JavaScript methods that can be called from BigQuery and `framework.wasm` which contains the MicroPython logic as WebAssembly. To initialize the MicroPython WebAssembly, I decided to write the bytes of `framework.wasm` into a byte array stored in a JavaScript file. This byte array is then accessed in `micropython.js`. It turned out that the size of the byte array exceeded the limit of 1 MB for external code files, so the byte array ended up to be split across multiple files, `part0.js` and `part1.js`, which are concatenated together in `micropython.js` and then used for initializing the MicroPython WebAssembly. 

The following code shows how an exemplary UDF that uses Micropython could look like:


```sql
CREATE TEMP FUNCTION
    pyFunc(x FLOAT64)
RETURNS STRING
LANGUAGE js AS """
    mp_js_init(64 * 1024);
    const pythonCode = `return sum(list(map(lambda x: x * 2, [1,2,3])))`;
    return mp_js_exec_str(pythonCode);
"""
OPTIONS (
    library = "gs://bucket/path/part0.js",
    library = "gs://bucket/path/part1.js",
    library = "gs://bucket/path/micropython.js"
);
SELECT
  pyFunc(x)
FROM (
  SELECT
    1 x,
    2 y)
```

The three files `part0.js`, `part1.js` and `micropython.js` need to be imported and therefore need to be stored on Google Cloud storage. In the JavaScript UDF, first MicroPython needs to be initialized by calling `mp_js_init` with the given stack size in bytes. Then Python code can be stored into a string and passed into `mp_js_exec_str` which runs the Python and returns the value of the last `return` statement as string.

The query can also be copied into the BigQuery Web console and executed there:

![Multiple search queries](/img/bigquery-screenshot.png)
*BigQuery web UI running Python code.*


## Try it out!

I created a [GitHub repository](https://github.com/scholtzan/python-udf-bigquery) with scripts to automatically generate and deploy UDFs containing Python code to BigQuery.