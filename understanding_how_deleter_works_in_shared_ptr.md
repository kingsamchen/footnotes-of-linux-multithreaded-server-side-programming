# Understanding How Deleters Work in Boost’s shared_ptr

Boost’s reference-counting smart pointer `shared_ptr` has the interesting characteristic that you can pass it a function or function object during construction, and it will invoke this `deleter` on the pointed-to object when the reference count goes to zero. At first blush, this seems unremarkable, but look at the code:

```c++
template<typename T>
class shared_ptr {
public:
        template<typename U, typename D>
        explicit shared_ptr(U* ptr, D deleter);
        ...
};
```

Notice that a `shared_ptr<T>` must somehow arrange for a deleter of type `D` to be invoked during destruction, yet the `shared_ptr<T>` has no idea what `D` is. The object can’t contain a data member of type `D`, nor can it point to an object of type `D`, because `D` isn’t known when the object’s data members are declared. So how can the `shared_ptr` object keep track of the deleter that will be passed in during construction and that must be used later when the `T` object is to be destroyed? More generally, how can a constructor communicate information of unknown type to the object it will be constructing, given that the object cannot contain anything referring to the type of the information?

The answer is simple: have the object contain a pointer to a base class of known type (Boost calls it `sp_counted_base`), have the constructor instantiate a template that derives from this base class using `D` as the instantiation argument (Boost uses the templates `sp_counted_impl_p` and `sp_counted_impl_pd`), and use a virtual function declared in the base class and defined in the derived class to invoke the deleter. (Boost uses `dispose`). Slightly simplified, it looks like this:

![](/img/shared_ptr_impl_diagram.gif)

It’s obvious—once you’ve seen it. But once you’ve seen it, you realize it can be used in all kinds of places, that it opens up new vistas for template design where templatized classes with relatively few template parameters (`shared_ptr` has only one) can reference unlimited amounts of information of types not known until later. Once I realized what was going on, I couldn’t help but smile and shake my head with admiration.

---

1. 见书 P20
2. 节选自 Scott Meyers 的 [My Most Important C++ Aha! Moments...Ever](https://www.artima.com/cppsource/top_cpp_aha_moments.html)
