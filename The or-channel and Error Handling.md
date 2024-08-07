---
id: or-channel
aliases:
  - The or-channel and Error Handling
tags:
  - golang
  - concurrency
---
# The or-channel

- This is a way for combining a done channels together into one place for closing them at once
- This pattern creates done channels through recursion
- This gives u a channel that will exited as soon as You have a closed channel among the channels or a written channel
- This creates a tree for the goroutines that can make the channel close as soon as one of the nodes is not there in other words closed
- This pattern is useful to employ at the intersection of modules in your system. At these intersections, you tend to have multiple conditions for canceling trees of gorou‐ tines through your call stack. Using the or function, you can simply combine these together and pass it down the stack.

**Example**
```go
var or func(channels ...<-chan interface{}) <-chan interface{}
or = func(channels ...<-chan interface{}) <-chan interface{} {
	switch len(channels) {
	case 0:
		return nil
	case 1:
		return channels[0]
	}
	orDone := make(chan interface{})
	go func() {
		defer close(orDone)
		switch len(channels) {
		case 2:
			select {
			case <-channels[0]:
			case <-channels[1]:
			}
		default:
			select {
			case <-channels[0]:
			case <-channels[1]:
			case <-channels[2]:
			case <-or(append(channels[3:], orDone)...):
			}
		}
	}()
	return orDone
}

```
# Error Handling

It is hard to handle error in concurrent programming. What we most of as do is just to print it to the stdout but instead what we can do it that we create a struct for the data that we share between the goroutines with an error field which will be used by the channels to send errors with the data that they have finished processing. 

**Example**:

```go
type Result struct {
	Error    error
	Response *http.Response
}
checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result {
	results := make(chan Result)
	go func() {
		defer close(results)
		for _, url := range urls {
			var result Result
			resp, err := http.Get(url)
			result = Result{Error: err, Response: resp}
			select {
			case <-done:
				return
			case results <- result:
			}
		}
	}()
	return results
}
done := make(chan interface{})
defer close(done)
urls := []string{"https://www.google.com", "https://badhost"}
for result := range checkStatus(done, urls...) {
	if result.Error != nil {
		fmt.Printf("error: %v", result.Error)
		continue
	}
	fmt.Printf("Response: %v\n", result.Response.Status)
}
```
