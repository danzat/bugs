# Who's calling?
## Introduction

In our product we place hooks on various system functions in order to modify existing application behaviour.

Regardless of the method of hooking, the basic concept is that suppose some system library implements function `F`, when that function is *hooked*, this means that if somewhere in the process some code calls `F`, it will end up reaching our own implementation of it which we will call `HOOK_F`. This implementation can wrap the original `F` with some code or might reimplement it completely.

This system allows us to implement specialized behaviours in processes but that is beyond the scope of this article.

Another thing about our system (and I believe any other system out there) are logs. In order for us to be able to debug and verify the functionality of our product we have logs places all over the place detailing, for example, the value of the parameters in the entrance of functions and return values in their end.

However, logs are something to be seen by the developers only, and they do add a bit of an overhead to the product, both in size (MiBs) and in performance, so we want to be able to easily switch them off.

To that end we have two build modes:
- Debug with all the logs
- Release where all the logs are shut out by the preprocessor, so they don't even get compiled

## The bug

## Investigation

I noticed that `dlsym(RTLD_NEXT, ...)` was constantly failing. But only in debug.
Well, it might just be that it's supposed to fail. To verify that I fired up `lldb` with the application in relese mode, and placed a conditional breakpoint on `dlsym` where the first parameter is `RTLD_NEXT`.

Surprise, dlsym returned a normal pointer and not `NULL`.

OK, so now I knew for certain that the failure was in `dlsym`, and it somehow failed due to the application being in debug.

So to understand what could possible make `dlsym(RTLD_NEXT, ...)` fail I went to look at dyld's dlsym code:

    void* dlsym(void* handle, const char* symbolName)
    {
        /* ... */
        // magic "search what I would see" handle
        if ( handle == RTLD_NEXT ) {
            void* callerAddress = __builtin_return_address(1); // note layers: 1: real client, 0: libSystem glue
            ImageLoader* callerImage = dyld::findImageContainingAddress(callerAddress);
            sym = callerImage->findExportedSymbolInDependentImages(underscoredName, dyld::gLinkContext, &image); // don't search image, but do search what it links against
            if ( sym != NULL ) {
                CRSetCrashLogMessage(NULL);
                return (void*)image->getExportedSymbolAddress(sym, dyld::gLinkContext);
            }
            const char* str = dyld::mkstringf("dlsym(RTLD_NEXT, %s): symbol not found", symbolName);
            dlerrorSet(str);
            free((void*)str);
            CRSetCrashLogMessage(NULL);
            return NULL;
        }
        /* ... */
    }

What caught my eye were these two lines:

    void* callerAddress = __builtin_return_address(1);
    ImageLoader* callerImage = dyld::findImageContainingAddress(callerAddress);

OK, so `dyld` figures who called it, and then fetches the correct "image" in which it would search the symbol.

So it makes sense why it should fail, now the call that came from library X is routed through our code, which to `dyld` it looks like our library made the `dlsym` call. So it fetches the wrong image, and can not find the symbol there, so it fails. Good.

Wait, not *good*. If this fails, why does it not fail all the time? After all, it does work in release. So what's the difference?

First, start with understanding how this `__builtin_return_address` works. From the GCC documentation (could not find the Clang docs):

> Built-in Function: `void * __builtin_return_address (unsigned int level)`
>   This function returns the return address of the current function, or of one of its callers. The level argument is number of frames to scan up the call stack. A value of 0 yields the return address of the current function, a value of 1 yields the return address of the caller of the current function, and so forth. When inlining the expected behavior is that the function returns the address of the function that is returned to. To work around this behavior use the noinline function attribute.
>   The level argument must be a constant integer.

Since we wanted to know what `dlsym` actually returned, the hook on it looked something like this:

    void * HOOK_dlsym(void * handle, const char * name)
    {
        void * ret = NULL;
        LOG("dlsym(%p, %s)", handle, name);
        if (/* handle and name match some criterion */)
        {
            /* Do some logic */
        }
        else
        {
            ret = dlsym(handle, name);
        }
        LOG("dlsym(%p, %s) = %p", handle, name, ret);
        return ret;
    }

So what interest me is that second if clause, what would it look like in ARM assembly?

    ldr r0, [sp, #handle]
    ldr r1, [sp, #name]
    bl dlsym
    str r0, [sp, #ret]
    ldr r0, [pc, #format_string] ; "dlsym(%p, %s) = %p"
    ldr r1, [sp, #handle]
    ldr r2, [sp, #name]
    ldr r3, [sp, #ret]
    bl LOG
    ldr r0, [sp, #ret]
    bx lr

How would that look in release?

    void * HOOK_dlsym(void * handle, const char * name)
    {
        void * ret = NULL;
        if (/* handle and name match some criterion */)
        {
            /* Do some logic */
        }
        else
        {
            ret = dlsym(handle, name);
        }
        return ret;
    }

So what interest me is that second if clause, what would it look like in ARM assembly?

    ldr r0, [sp, #offset_of_saved_handle]
    ldr r1, [sp, #offset_of_saved_name]
    b.w dlsym

Notice how in debug `dlsym` was called with a branch-and-link (`bl`) while in debug it was just a regular branch (`b.w`)? This is called tail-call optimization. This happens when the last statement in a function is another function call, so that function can be in charge of *returning* to the caller.

So if say we have the following call graph:

picture of F1 -> F2 -> F3



## Summary

## Conclusions
