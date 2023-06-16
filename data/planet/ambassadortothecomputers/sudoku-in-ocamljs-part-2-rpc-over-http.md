---
title: 'Sudoku in ocamljs, part 2: RPC over HTTP'
description: Last time  we made a simple user interface for Sudoku with the Dom  module
  of ocamljs . It isn't a very fun game though since there are no p...
url: http://ambassadortothecomputers.blogspot.com/2009/05/sudoku-in-ocamljs-part-2-rpc-over-http.html
date: 2009-05-04T05:35:00-00:00
preview_image:
featured:
authors:
- ambassadortothecomputers
---

<p><a href="http://ambassadortothecomputers.blogspot.com/2009/04/sudoku-in-ocamljs-part-1-dom.html">Last time</a> we made a simple user interface for Sudoku with the <code>Dom</code> module of <a href="http://code.google.com/p/ocamljs">ocamljs</a>. It isn't a very fun game though since there are no pre-filled numbers to constrain the board. So let's add a button to get a new game board; here's the <a href="http://orpc2.googlecode.com/svn/examples/sudoku/index.html">final result</a>.<br/>
</p><p>I don't know much about <a href="http://en.wikipedia.org/wiki/Algorithmics_of_sudoku">generating Sudoku boards</a>, but it seems like it might be slow to do it in the browser, so we'll do it on the server, and communicate to the server with OCaml function calls using the RPC over HTTP support in <a href="http://code.google.com/p/orpc2">orpc</a>.<br/>
</p><b>The 5-minute monad</b><br/>
<p>But first I'm going to give you a brief introduction to <em>monads</em> (?!). Bear with me until I can explain why we need monads for Sudoku, or skip it if this is old hat to you. We'll transform the following fragment into monadic form: </p><pre><span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">foo</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">()</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">=</span> 7 <span class="htmlize-tuareg-font-lock-governing">in</span>
bar <span class="htmlize-tuareg-font-lock-operator">(</span>foo <span class="htmlize-tuareg-font-lock-operator">())</span>
</pre>First put it in <em>named form</em> by <code>let</code>-binding the result of the nested function application: <pre><span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">foo</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">()</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">=</span> 7 <span class="htmlize-tuareg-font-lock-governing">in</span>
<span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-variable-name">f </span><span class="htmlize-tuareg-font-lock-operator">=</span> foo <span class="htmlize-tuareg-font-lock-operator">()</span> <span class="htmlize-tuareg-font-lock-governing">in</span>
bar f
</pre>Then introduce two new functions, <code>return</code> and <code>bind</code>: <pre><span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">return</span><span class="htmlize-variable-name"> x </span><span class="htmlize-tuareg-font-lock-operator">=</span> x
<span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">bind</span><span class="htmlize-variable-name"> x f </span><span class="htmlize-tuareg-font-lock-operator">=</span> f x

<span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">foo</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">()</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">=</span> return 7 <span class="htmlize-tuareg-font-lock-governing">in</span>
bind <span class="htmlize-tuareg-font-lock-operator">(</span>foo <span class="htmlize-tuareg-font-lock-operator">())</span> <span class="htmlize-tuareg-font-lock-operator">(</span><span class="htmlize-keyword">fun</span> <span class="htmlize-variable-name">f </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span>
  bar f<span class="htmlize-tuareg-font-lock-operator">)</span>
</pre>These functions are a bit mysterious (although the name &quot;bind&quot; is suggestive of <code>let</code>-binding), but we haven't changed the meaning of the fragment. Next we would like to enforce that the only way to use the result of <code>foo ()</code> is by calling <code>bind</code>. We can do that with an abstract type: <pre><span class="htmlize-tuareg-font-lock-governing">type</span> <span class="htmlize-tuareg-font-lock-operator">'</span><span class="htmlize-type">a t</span>
<span class="htmlize-tuareg-font-lock-governing">val</span> <span class="htmlize-variable-name">return </span><span class="htmlize-tuareg-font-lock-operator">:</span> <span class="htmlize-tuareg-font-lock-operator">'</span><span class="htmlize-type">a </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span><span class="htmlize-type"> </span><span class="htmlize-tuareg-font-lock-operator">'</span><span class="htmlize-type">a t</span>
<span class="htmlize-tuareg-font-lock-governing">val</span> <span class="htmlize-variable-name">bind  </span><span class="htmlize-tuareg-font-lock-operator">:</span> <span class="htmlize-tuareg-font-lock-operator">'</span><span class="htmlize-type">a t </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span><span class="htmlize-type"> </span><span class="htmlize-tuareg-font-lock-operator">('</span><span class="htmlize-type">a </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span><span class="htmlize-type"> </span><span class="htmlize-tuareg-font-lock-operator">'</span><span class="htmlize-type">b t</span><span class="htmlize-tuareg-font-lock-operator">)</span><span class="htmlize-type"> </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span><span class="htmlize-type"> </span><span class="htmlize-tuareg-font-lock-operator">'</span><span class="htmlize-type">b t</span>
</pre>Taking <code>type 'a t = 'a</code>, the definitions of <code>return</code> and <code>bind</code> match this signature. So what have we accomplished? We've abstracted out the notion of <em>using the result of a computation</em>. It turns out that there are many useful structures matching this signature (and satisfying <a href="http://www.google.com/search?q=monad%20laws">some equations</a>), called monads. It's convenient that they all match the same signature, in part because we can mechanically convert ordinary code into monadic code, as we've done here, or even use a <a href="http://www.cas.mcmaster.ca/~carette/pa_monad/">syntax extension</a> to do it for us.<br/>
<b>Lightweight threads in Javascript</b><br/>
<p>One such useful structure is the <a href="http://ocsigen.org/lwt">Lwt</a> library for cooperative threads. You can write Lwt-threaded code by taking ordinary threaded code and converting it to monadic style. In Lwt, <code>'a t</code> is the type of threads returning <code>'a</code>. Then <code>bind t f</code> calls <code>f</code> on the value of the thread <code>t</code> <em>once <code>t</code> has finished</em>, and <code>return x</code> is an already-finished thread with value <code>x</code>.<br/>
</p><p>Lwt threads are cooperative: they run until they complete or block waiting on the result of another thread, but aren't ever preempted. It can be easier to reason about this kind of threading, because until you call <code>bind</code>, there's no possibility of another thread disturbing any state you're working on.<br/>
</p><p>Lwt threads are a great match for Javascript, which doesn't have preemptive threads (although plugins like <a href="http://gears.google.com/">Google Gears</a> provide them), because they need no special support from the language except closures. Typically in Javascript you write a blocking computation as a series of callbacks. You're doing essentially the same thing with Lwt, but it's packaged up in a clean interface.<br/>
</p><b>Orpc for RPC over HTTP</b><br/>
<p>The reason we care about threads in Javascript is that we want to make a blocking RPC call to the server to retrieve a Sudoku game board, without hanging the browser. We'll use orpc to generate stubs for the client and server. In the client the call returns an Lwt thread, so you need to call <code>bind</code> to get the result. In the server it arrives as an ordinary procedure call.<br/>
</p><p>To use orpc you write down the signature of the RPC interface, in <code>Lwt</code> and <code>Sync</code> forms for the client and server. Orpc checks that the two forms are compatible, and generates the stubs. Here's our interface (<a href="http://code.google.com/p/orpc2/source/browse/trunk/examples/sudoku/proto.ml">proto.ml</a>): </p><pre><span class="htmlize-tuareg-font-lock-governing">module</span> <span class="htmlize-tuareg-font-lock-governing">type</span> <span class="htmlize-type">Sync </span><span class="htmlize-tuareg-font-lock-operator">=</span>
<span class="htmlize-tuareg-font-lock-governing">sig</span>
  <span class="htmlize-tuareg-font-lock-governing">val</span> <span class="htmlize-variable-name">get_board </span><span class="htmlize-tuareg-font-lock-operator">:</span> <span class="htmlize-type">unit </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span><span class="htmlize-type"> int option array array</span>
<span class="htmlize-tuareg-font-lock-governing">end</span>

<span class="htmlize-tuareg-font-lock-governing">module</span> <span class="htmlize-tuareg-font-lock-governing">type</span> <span class="htmlize-type">Lwt </span><span class="htmlize-tuareg-font-lock-operator">=</span>
<span class="htmlize-tuareg-font-lock-governing">sig</span>
  <span class="htmlize-tuareg-font-lock-governing">val</span> <span class="htmlize-variable-name">get_board </span><span class="htmlize-tuareg-font-lock-operator">:</span> <span class="htmlize-type">unit </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span><span class="htmlize-type"> int option array array Lwt.t</span>
<span class="htmlize-tuareg-font-lock-governing">end</span>
</pre>The <code>get_board</code> function returns a 9x9 array, each cell of which may contain <code>None</code> or <code>Some k</code> where <code>k</code> is 1 to 9. We can't capture all these constraints in the type, but we get more static checking than if we were passing JSON or XML.<br/>
<b>Generating the board</b><br/>
<p>On the <a href="http://code.google.com/p/orpc2/source/browse/trunk/examples/sudoku/server.ml">server</a>, we implement a module that matches the <code>Sync</code> signature. (You can see that I didn't actually implement any Sudoku-generating code, but took some fixed examples from Gnome Sudoku.) Then there's some boilerplate to set up a Netplex HTTP server and register the module at the <code>/sudoku</code> path. It's pretty simple. The <code>Proto_js_srv</code> module contains stubs generated by orpc from <code>proto.ml</code>, and <code>Orpc_js_server</code> is part of the orpc library.<br/>
</p><b>Using the board</b><br/>
<p>The <a href="http://code.google.com/p/orpc2/source/browse/trunk/examples/sudoku/sudoku.ml">client</a> is mostly unchanged from last time. There's a new button, &quot;New game&quot;, that makes the RPC call, then fills in the board from the result. </p><pre><span class="htmlize-tuareg-font-lock-governing">let</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">(&gt;&gt;=)</span><span class="htmlize-variable-name"> </span><span class="htmlize-tuareg-font-lock-operator">=</span> <span class="htmlize-type">Lwt</span>.<span class="htmlize-tuareg-font-lock-operator">(&gt;&gt;=)</span>
</pre>The <code>&gt;&gt;=</code> operator is another name for <code>bind</code>. If you aren't using <a href="http://www.cas.mcmaster.ca/~carette/pa_monad/">pa_monad</a> (which we aren't here), it makes a sequence of <code>bind</code>s easier to read. <pre><span class="htmlize-tuareg-font-lock-governing">module</span> <span class="htmlize-type">Server </span><span class="htmlize-tuareg-font-lock-operator">=</span>
  <span class="htmlize-type">Proto_js_clnt</span>.Lwt<span class="htmlize-tuareg-font-lock-operator">(</span><span class="htmlize-tuareg-font-lock-governing">struct</span>
    <span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">with_client</span><span class="htmlize-variable-name"> f </span><span class="htmlize-tuareg-font-lock-operator">=</span> f <span class="htmlize-tuareg-font-lock-operator">(</span><span class="htmlize-type">Orpc_js_client</span>.create <span class="htmlize-string">&quot;/sudoku&quot;</span><span class="htmlize-tuareg-font-lock-operator">)</span>
  <span class="htmlize-tuareg-font-lock-governing">end</span><span class="htmlize-tuareg-font-lock-operator">)</span>
</pre>This sets up the RPC interface, so calls on the <code>Server</code> module become RPC calls to the server. The <code>Proto_js_client</code> module contains stubs generated from <code>proto.ml</code>, and <code>Orpc_js_client</code> is part of the orpc library. (In the actual source you'll see that I faked this out in order to host the running example on Google Code--there's no way to run an OCaml server, so I randomly choose a canned response.) <pre><span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-function-name">get_board</span><span class="htmlize-variable-name"> rows _ </span><span class="htmlize-tuareg-font-lock-operator">=</span>
  ignore
    <span class="htmlize-tuareg-font-lock-operator">(</span><span class="htmlize-type">Server</span>.get_board <span class="htmlize-tuareg-font-lock-operator">()</span> <span class="htmlize-tuareg-font-lock-operator">&gt;&gt;=</span> <span class="htmlize-keyword">fun</span> <span class="htmlize-variable-name">board </span><span class="htmlize-tuareg-font-lock-operator">-&gt;</span>
      <span class="htmlize-keyword">for</span> i <span class="htmlize-tuareg-font-lock-operator">=</span> 0 <span class="htmlize-keyword">to</span> 8 <span class="htmlize-keyword">do</span>
        <span class="htmlize-keyword">for</span> j <span class="htmlize-tuareg-font-lock-operator">=</span> 0 <span class="htmlize-keyword">to</span> 8 <span class="htmlize-keyword">do</span>
          <span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-variable-name">cell </span><span class="htmlize-tuareg-font-lock-operator">=</span> rows.<span class="htmlize-tuareg-font-lock-operator">(</span>i<span class="htmlize-tuareg-font-lock-operator">)</span>.<span class="htmlize-tuareg-font-lock-operator">(</span>j<span class="htmlize-tuareg-font-lock-operator">)</span> <span class="htmlize-tuareg-font-lock-governing">in</span>
          <span class="htmlize-tuareg-font-lock-governing">let</span> <span class="htmlize-variable-name">style </span><span class="htmlize-tuareg-font-lock-operator">=</span> cell<span class="htmlize-tuareg-font-lock-operator">#</span>_get_style <span class="htmlize-tuareg-font-lock-governing">in</span>
          style<span class="htmlize-tuareg-font-lock-operator">#</span>_set_backgroundColor <span class="htmlize-string">&quot;#ffffff&quot;</span><span class="htmlize-tuareg-font-lock-operator">;</span>
          <span class="htmlize-keyword">match</span> board.<span class="htmlize-tuareg-font-lock-operator">(</span>i<span class="htmlize-tuareg-font-lock-operator">)</span>.<span class="htmlize-tuareg-font-lock-operator">(</span>j<span class="htmlize-tuareg-font-lock-operator">)</span> <span class="htmlize-keyword">with</span>
            <span class="htmlize-tuareg-font-lock-operator">|</span> None <span class="htmlize-tuareg-font-lock-operator">-&gt;</span>
                cell<span class="htmlize-tuareg-font-lock-operator">#</span>_set_value <span class="htmlize-string">&quot;&quot;</span><span class="htmlize-tuareg-font-lock-operator">;</span>
                cell<span class="htmlize-tuareg-font-lock-operator">#</span>_set_disabled <span class="htmlize-constant">false</span>
            <span class="htmlize-tuareg-font-lock-operator">|</span> Some n <span class="htmlize-tuareg-font-lock-operator">-&gt;</span>
                cell<span class="htmlize-tuareg-font-lock-operator">#</span>_set_value <span class="htmlize-tuareg-font-lock-operator">(</span>string_of_int n<span class="htmlize-tuareg-font-lock-operator">);</span>
                cell<span class="htmlize-tuareg-font-lock-operator">#</span>_set_disabled <span class="htmlize-constant">true</span>
        <span class="htmlize-keyword">done</span>
      <span class="htmlize-keyword">done</span><span class="htmlize-tuareg-font-lock-operator">;</span>
      <span class="htmlize-type">Lwt</span>.return <span class="htmlize-tuareg-font-lock-operator">());</span>
  <span class="htmlize-constant">false</span>
</pre>This is the event handler for the &quot;New game&quot; button. We call <code>get_board</code>, <code>bind</code> the result, then fill in the board. If there's a number in a cell we disable the input box so the player can't change it. Here's the <a href="http://code.google.com/p/orpc2/source/browse/trunk/examples/sudoku">full code</a>.<br/>
<p>Doing AJAX programming with orpc and Lwt really shows off the power of compiling OCaml to Javascript. While <a href="http://code.google.com/webtoolkit/">Google Web Toolkit</a> has a similar RPC mechanism (that generates stubs from Java interfaces), it's much clumsier to use, because you're still working at the level of callbacks rather than threads. Maybe you could translate Lwt to Java, but it would be painfully verbose without type inference.<br/>
</p><p>This monad stuff will come in handy again next time, when we'll revisit the problem of checking the Sudoku constraints on the board, using <a href="http://code.google.com/p/froc">froc</a>.<br/>
</p>