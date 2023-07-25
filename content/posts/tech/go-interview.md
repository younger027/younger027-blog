1.go-channel阻塞的场景

- 无缓冲的channel，同一个协程内读写，会导致all goroutine are asleep.dead lock
- 无缓冲的channel，通道的同步写早于读channel
- 从一个没有数据的channel里拿数据引起的死锁
- 循环等待引起的死锁，两个G互相持有对方拥有的资源，无法读写
- 有缓冲区，收发在同一个G，但是缓冲区已满，写阻塞
- 有缓冲区，读空的channel，读阻塞，可以加select控制

2.读写channel哪个先关。

关闭channel的原则，不要让receiver来关闭chan。也不要在多个sender的时候由sender关闭chan。会导致panic的情况有两个。一个是给已经关闭的chan写数据。另一个是重复close chan会导致panic。一些常见的方式，可以使用sync.Once和sync.mutex来关闭chan。还可以直接用panic和recovery来处理panic。

一读一写(只要一个写都可以写端关闭)。写端关闭。不写数据的时候，关闭chan，通知读端。

一个读多个写。读端通知写端关闭。可以新加个close chan，通知写端不要输入了。

多读多写。增加toStop chan去通知关闭close chan。读写端会有select检查close chan的状态。

```go
func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// ...
	const MaxRandomNumber = 100000
	const NumReceivers = 10
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})
	// stopCh is an additional signal channel.
	// Its sender is the moderator goroutine shown below.
	// Its reveivers are all senders and receivers of dataCh.
	toStop := make(chan string, 1)
	// the channel toStop is used to notify the moderator
	// to close the additional signal channel (stopCh).
	// Its senders are any senders and receivers of dataCh.
	// Its reveiver is the moderator goroutine shown below.

	var stoppedBy string

	// moderator
	go func() {
		stoppedBy = <-toStop // part of the trick used to notify the moderator
		// to close the additional signal channel.
		close(stopCh)
	}()

	// senders
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				Val := rand.Intn(MaxRandomNumber)
				if Val == 0 {
					// here, a trick is used to notify the moderator
					// to close the additional signal channel.
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}

				// the first select here is to try to exit the
				// goroutine as early as possible.
				select {
				case <-stopCh:
					return
				default:
				}

				select {
				case <-stopCh:
					return
				case dataCh <- Val:
				}
			}
		}(strconv.Itoa(i))
	}

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			defer wgReceivers.Done()

			for {
				// same as senders, the first select here is to
				// try to exit the goroutine as early as possible.
				select {
				case <-stopCh:
					return
				default:
				}

				select {
				case <-stopCh:
					return
				case Val := <-dataCh:
					if Val == MaxRandomNumber-1 {
						// the same trick is used to notify the moderator
						// to close the additional signal channel.
						select {
						case toStop <- "receiver#" + id:
						default:
						}
						return
					}

					log.Println(Val)
				}
			}
		}(strconv.Itoa(i))
	}

	// ...
	wgReceivers.Wait()
	log.Println("stopped by", stoppedBy)
}

```



