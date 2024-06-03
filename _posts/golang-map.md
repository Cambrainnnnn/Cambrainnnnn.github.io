> Yes, maps that shrink permanently currently never get cleaned up after. As usual, the implementation challenge is with iterators. <br>
> 实现 map 收缩最大的挑战在于 iterator.



1. [Iterating over maps in Go - Medium](https://medium.com/i0exception/map-iteration-in-go-275abb76f721)
2. [runtime: shrink map as elements are deleted - github](https://github.com/golang/go/issues/20135)