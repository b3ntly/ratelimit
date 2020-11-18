Ratelimit is an implementation of a [Leaky Bucket algorithm](https://en.wikipedia.org/wiki/Leaky_bucket) and supports a pluggable backend so you may use different storage mechanisms such a redis, in-memory, etc.

Leaky Bucket Algorithms are useful for ratelimiting applications. For background 
information I recommend you read [this](https://stripe.com/blog/rate-limiters) article from stripe.

### Basic usage

I recommend implementing your own ratelimit.Backend interface even though implementations and examples exist in ratelimit/redigo, ratelimit/radix, and ratelimit/memory packages and respective test files. This is because the connection pooling for your database of choice will have a large impact on your application performance, given that for most web applications your ratelimiter could make several calls per request.

```
package main

import (
	"fmt"
	"strconv"
	"time"

	"github.com/gomodule/redigo/redis"
	"github.com/b3ntly/ratelimit"
)

// Backend implements the ratelimit.Backend interface
type Backend struct {
	*redis.Pool
}

// GetState retrieves allowance and lastAccessedTimestampNS from a hash set at key
func (b *Backend) GetState(key string) (allowance int64, lastAccessedTimestampNS int64, err error) {
	c := b.Get() 
	defer c.Close() 
	
	m, err := redis.Int64Map(c.Do("HGETALL", key))
	if err != nil {
		return 0, 0, err
	}

	allowance, allowanceExists := m["0"]
	lastAccessedTimestampNS, lastAccessedTimestampNSExists := m["1"]
	if !allowanceExists || !lastAccessedTimestampNSExists {
		return 0, 0, nil
	}

	return allowance, lastAccessedTimestampNS, nil
}

// SetState sets allowance and lastAccessedTimestampNS as a hash set using they keys 0 and 1 to reduce
// size, respectively
func (b *Backend) SetState(key string, allowance int64, lastAccessedTimestampNS int64) error {
	c := b.Get() 
	defer c.Close() 
	
	if _, err := c.Do("HSET", key, "0", strconv.FormatInt(allowance, 10), "1", strconv.FormatInt(lastAccessedTimestampNS, 10)); err != nil {
		return err
	}

	return nil
}

func main() {
	pool := &redis.Pool{
		Wait:            true,
		MaxIdle:         1,
		MaxActive:       100,
		IdleTimeout:     time.Millisecond * 500,
		MaxConnLifetime: time.Millisecond * 500,
		Dial: func() (redis.Conn, error) {
			return redis.Dial("tcp", "localhost:6379", redis.DialPassword("password"), redis.DialDatabase(0))
		},
	}

	backend := &Backend{
		pool,
	}

	rate := int64(1)
	interval := time.Second 
	burst := int64(10) 
	limiter := ratelimit.New(rate, interval, burst, backend)

	successes := 0
	failures := 0 
	for i := 0; i < 15; i++ {
		wait, err := limiter.Allow("benjamin")
		if err != nil {
			println(err.Error())
			return 
		}

		if wait == 0 {
			successes++
			continue 
		}

		failures++ 
		time.Sleep(wait)
	}

	fmt.Printf("successes: %v\n", successes)
	fmt.Printf("failures: %v\n", failures)
}
```

### Weaknesses 

* This library uses a simple configuration for the rate, burst, and interval, properties of the algorithm. In production applications you will likely desire per-user configuration. For example, Amy pays $5 for your api and should have 5 requests per second, while George pays $10 and should have 10 requests per second.

* Testing could be more rigorous especially for concurrent use cases and dynamically changing configuration. See ratelimit_test::refillAllowanceTests for basic examples that are testing.

* Convenience methods for logging, per-user overrides (i.e. ResetRateLimitForKey()), and other helpers would be useful for a production deployment.