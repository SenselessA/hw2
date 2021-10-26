Рефакторинг 

```
package main

import (
	"fmt"
	"log"
	"math/rand"
	"os"
	"sync"
	"time"
)

var actions = []string{
	"logged in",
	"logged out",
	"created record",
	"deleted record",
	"updated account",
}

type logItem struct {
	action    string
	timestamp time.Time
}

type User struct {
	id    int
	email string
	logs  []logItem
}

func (u User) getActivityInfo() string {
	output := fmt.Sprintf("UID: %d; Email: %s;\nActivity Log:\n", u.id, u.email)
	for index, item := range u.logs {
		output += fmt.Sprintf("%d. [%s] at %s\n", index+1, item.action, item.timestamp.Format(time.RFC3339))
	}
	return output
}

func main() {
	rand.Seed(time.Now().Unix())

	startTime := time.Now()

	wg := &sync.WaitGroup{}
	usersCount, workerCount := 100, 100
	jobs := make(chan int, usersCount)
	users := make(chan User, usersCount)

	for i := 0; i < workerCount; i++ {
		go generateUsers(jobs, users)
	}

	for i := 0; i < workerCount; i++ {
		go saveUserInfo(users, wg)
	}

	for j := 0; j < usersCount; j++ {
		wg.Add(1)
		jobs <- j + 1
	}

	close(jobs)
	wg.Wait()

	fmt.Printf("DONE! Time Elapsed: %.2f seconds\n", time.Since(startTime).Seconds())
}

func saveUserInfo(users <-chan User, wg *sync.WaitGroup) {

	for user := range users {
		fmt.Printf("WRITING FILE FOR UID %d\n", user.id)

		filename := fmt.Sprintf("users/uid%d.txt", user.id)
		file, err := os.OpenFile(filename, os.O_RDWR|os.O_CREATE, 0644)
		if err != nil {
			log.Fatal(err)
		}

		_, err = file.WriteString(user.getActivityInfo())
		if err != nil {
			log.Fatal(err)
		}

		wg.Done()
	}
}

func generateUsers(jobs <-chan int, users chan<- User) {
	for i := range jobs {
		users <- User{
			id:    i,
			email: fmt.Sprintf("user%d@company.com", i),
			logs:  generateLogs(rand.Intn(1000)),
		}
		fmt.Printf("generated user %d\n", i)
		time.Sleep(time.Millisecond * 100)
	}
}

func generateLogs(count int) []logItem {
	logs := make([]logItem, count)

	for i := 0; i < count; i++ {
		logs[i] = logItem{
			action:    actions[rand.Intn(len(actions)-1)],
			timestamp: time.Now(),
		}
	}

	return logs
}


```