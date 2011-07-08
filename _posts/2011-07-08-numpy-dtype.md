--- 
title: The power of numpy dtype keyword
layout: post
---

One of Numpy very usefull feature is its ability to define very easily
an array with a very large choice of data types. For instance, this
gives the possibility to very easily test an algorithm for its
robustness against error propagation due to the floating point
precision. You could write something like this :

{% highlight python %}
# generate arrays with different data types
ashape = (16,)
dtypes = (np.float16, np.float32, np.float64)
arrays = [np.ones(ashape, dtype=dtype) for dtype in dtypes]
# test algorithm against different data types
out_arrays = [my_algorithm(arr) for arr in arrays]
# measure the difference
print("cumulated difference 16bits vs 64 bits %f" % np.sum(np.abs(out_arrays[0] - out_arrays[-1])))
print("cumulated difference 32bits vs 64 bits %f" % np.sum(np.abs(out_arrays[1] - out_arrays[-1])))
{% endhighlight %}

Changing the data dtype from 64 bits to 32 bits can save your code a
factor of 2 in memory usage. This can be very important for data
intensive applications. Numpy even provide a 16 bits float which can
be useful if precision requirements are not very high.
