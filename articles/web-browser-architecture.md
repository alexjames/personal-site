# Web Browser Architecture

No discussion of web technologies is complete without understanding the internals of a web browser. In some sense, the browser is the operating system of the internet, for a large part of your web experience is facilitated by it. Under the hood, modern browsers are fairly complex beasts. It is important for anyone trying to develop a deep understanding of the web to form an intuitive understanding of what happens under the hood when you hit enter a web URL.

We’ll take a look at the high level steps that a typical browser might take to display a page. The concepts we discuss are fairly browser-agnostic, though specific implementation details may vary across browser versions and platforms. 

For illustrative purposes, let’s take a look at a simple HTML file.

```
<html>
  <body>
    <p>
      Hello there!
    </p>
    <div> 
      <img src="hello.jpg"/>
    </div>
  </body>
</html>
```

The HTML body contains a <p> tag and an <img> tag nested within a <div>. When rendered by the browser, we should see the text “Hello there!” followed by an image.

To make things more interesting, let’s add a <head> tag. Within that, we’ll introduce a simple script that modifies the text on the page. This is done by modifying the <p> tag that originally appeared in the html. We’ll also include some CSS styling information for the whole page. The final HTML will look as follows.

```
<html>
  <head>
    <script>
      window.onload = function() {
        const text_box = document.getElementsByTagName("p")[0];
        text_box.innerHTML = "Welcome!";
     }
    </script>
    <style>
      body {
        background-color:lightgrey;
      }
      p {
        color: blue;
        font-family: verdana;
        font-size: 200%;
      }
    </style>
  </head>
  <body>
    <p>
      Hello there!
    </p>
    <div> 
      <img src="hello.jpg"/>
    </div>
  </body>
</html>
```

We’ll assume that this HTML is being served on the domain “www.example.com”. You can access this page by typing in the domain name in any standard web browser’s URL bar. Here is what the final page would appear as -

Architecture
A web browser in its simplest form is an HTTP client. It makes requests to web servers using the HTTP protocol and retrieves the responses. It then processes those responses to display the corresponding data in the browser window.

Figure x shows a reference architecture [3] for components of a web browser.
The user interface consists of everything that is visible to a user within the browser window.
The rendering engine parses and lays out the web document.
The networking engine encompasses all networking functionality.
The javascript interpreter executes scripts contained on a page.
The UI backend is tasked with the actual drawing on the window. 
The browser engine provides an interface into the rendering engine.
A data persistence layer handles cookies, caches and bookmarks.

These are the five high-level steps for displaying the page in the browser:
1. The document is fetched.
2. The document is parsed
3. A DOM (Document Object Model) tree and styling tree are constructed
4. The layout of the document is determined
5. Painting (or rasterizing) of the document on-screen

Let’s look at each of these steps in greater detail.

### Fetch
When you enter an address into your browser, it makes an HTTP GET request to the web server at the address specified by the URL. The request might look as follows.

```
GET / HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.69 Safari/537.36
Connection: keep-alive
```

The User-Agent header identifies the browser to the server. This particular request originates from the Chrome browser [7]. The Connection header specifies the intent to maintain a persistent TCP connection for multiple HTTP requests.

The web server responds back with an HTTP 200 status and the web page that has been requested.

```
HTTP/1.1 200 OK
Date: Tue, 20 Aug 2019 03:45:34 GMT
Content-Length: 516
Content-Type: text/html; charset=UTF-8
Server: Apache/2.4.1 (Unix)
<head>
  <body>
...
```

The Content-Type header instructs the browser how to interpret the data portion of the HTTP response. "text/html; charset=UTF-8"  basically means that the data being sent back is an HTML document using UTF-8 character encoding. Content-Length specifies the size of the document data. The browser strips the data from the response message to acquire the actual web page.

A browser might also add the Accept-Encoding header with values like “gzip, deflate”
the request. This tells the web server that it supports data compression. In such a case, the server might decide to compress the data before it sends it. This allows for less data to be sent over the wire, with a little extra work required by both the server and the browser. Since the cost of network communication tends to add the biggest delay while processing a webpage, the lesser the bytes that need to be sent, the sooner the browser can receive the data and work on displaying it.

### Parse
Now that the browser has the web page, it can start parsing the document. Parsing is the process of reading the HTML data and translating it into a representation that the browser understands. This typically means constructing one or more data structures (or “trees”) that help the browser understand the data.

Parsing is done in two parts. The lexical analyzer (lexer or tokenizer) phase breaks down the data into individual tokens. Examples of tokens are start tags, end tags, attribute names, attribute values, etc. The parser then uses syntactical rules of the HTML language to create the Document Object Model (DOM) of the document. This is represented as a logical tree. More on this in the next section.

During parsing, the browser might notice that the page references other resources such as images, scripts or documents that need to be fetched. These are fetched in parallel to speed up the processing of the web page. In our case, when the <img> tag is encountered, a new HTTP request is made to fetch the "example.jpg" image file. The browser might assign this task to a separate browser worker thread.

By default, script execution is parser blocking. This means that when a <script> tag is encountered, the browser will pause the parsing of the document and call upon the Javascript interpreter to run the code between <script> tags. Further parsing is only resumed once script execution is complete. The script from our example sets up a custom handler to the window.onload property. This indicates to the browser that the handler code is meant to be executed once the whole webpage has been loaded.
Tree Construction
The DOM is an "object" representation (in the programming language sense) of all the elements and content in a web document. It provides an interface for programming languages (say Javascript) to interact with and manipulate the document. This allows web pages to be dynamic and interactive. The DOM represents a document as a logical tree and its generation is specified in the HTML standard [6]. You can use the Inspect option in Chrome to look at the DOM tree of a web page.

The DOM tree consists of nodes that represent the parts of the document. Every tag in the  HTML ends up being stored hierarchically as a node in the tree. The browser does not necessarily need to store the data internally as a tree, as long as it exposes an interface consistent with the DOM interface. The DOM tree created from our example file looks like this -




The browser also parses the style information from CSS styling sheets and tag attributes to generate separate styling trees. The nodes in the styling tree may or may not have 1:1 correspondence with the nodes of the DOM tree. Here is an example of what that might look like for our example HTML.




### Layout
By the end of the previous stage, the browser has figured out the elements of the document and some of their visual characteristics. It can now use this information to compute the layout of each visible element on the screen. The layout step determines the geometry and position of every object that is to be rendered. The layout process is also known as the reflow process. 


The browser has to maintain the positions of each element with respect to each other. Since this stage can be a computationally expensive process, the browser applies several optimizations here. For instance, it may group elements whose positions affect each other. This way, if the position of one of the elements changes, it only has to update the group and not recompute the layout of every element on the page.

### Paint
This is the final phase of displaying a page when the actual pixels are painted on-screen. This involves displaying every visual aspect of an element such as its color, size, text, borders etc.

Objects are rendered in the order in which they should be displayed, from objects in the back to those in the front. A z-index might be set for each element to determine the order in which they will be drawn. For instance, the background will be drawn before the overlaying text and divs.

The browser may make use of the GPU to accelerate the drawing process. As this stage is expensive, it employs several tricks here to speed up the process. Slow rendering can significantly affect user attention. A browser may try to batch paint operations and minimize the number of repaints it needs to keep the web page responsive. On changes to the DOM, rather than recalculating the whole layout and re-painting everything, it will try to only re-paint the portions on the screen that actually change. 

Elements hidden by larger objects may not be painted at all. Larger web pages that don’t fit on the screen will have portions not actively visible. The browser may choose to only render those parts if the user scrolls to those regions.

By the end of this stage, we will have a fully rendered web page in our browser. Or will we?

### Interactivity
In recent years, it’s more common to have web pages that are interactive. This interactivity is brought in part through languages such as Javascript, that can access and modify the DOM. Whenever this happens, the browser has to update parts of the DOM trees. Layouts may change and parts of the screen may need to be redrawn.

In our demo page, we have a script that modifies the paragraph tag and changes the text to “Welcome”. In the initial rendering, the browser might display “Hello there!”. After the page is loaded, the script handler is executed which updates the innerHTML of the <p> tag. This re-engages the browser to update this part of the page, replace the text and redraw it on the screen. This update might happen in the blink of an eye and may not be noticeable.