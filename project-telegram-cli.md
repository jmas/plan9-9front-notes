# Project for Plan 9 / 9front: CLI Telegram client

I love to chat with friends with Telegram. I know that we have IRC, but TG is modern way of communication. So I need to have ability to check new messages or write quick response to Telegram.

I found that we have working telegram library written on Go [gotd/td](https://github.com/gotd/td). So I tested it on my Mac and it's working. I worote small CLI program using Gemini, lets say PoC. Main question is it able to compile and run on Plan9 / 9front? I'm waiting for Asus Eee PC 1000px for 9front installation. When I get it and put 9front on it I'll first of all compile this code. It will be surprise for me if it runs well.

I have thought about how to use this CLI client. I need to research how 9front's IRQ CLI client is working and steal CLI interface from it. For example, I could run `tgc -L` for get list of chats. Then run `tgc -C <ID of Chat>` for list latest messages from the chat. Then if I want to send message to chat I can write `echo "New message" > tgc -C <ID of Chat>` and message will be sent to the chat. Also we should able to run `tgc -I` in interactive mode: I select chat, then see updates, send message or back to chat list. Auth will be triggered by `tgc -A` and it will create or update `session.json` file.

## Build instructions

1. Create an app via link: [my.telegram.org/apps](https://my.telegram.org/apps)
2. Init Go project `go mod init go-tg-client`
3. Set `TG_API_ID`, `TG_API_HASH`, `TG_PHONE` in `.env`
4. Put source to `main.go`
5. Install deps `go mod tidy`
6. Run `go run main.go`

## Code

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/joho/godotenv"
	"go.uber.org/zap"

	"github.com/gotd/td/session"
	"github.com/gotd/td/telegram"
	"github.com/gotd/td/telegram/auth"
	"github.com/gotd/td/tg"
)

func scanString(msg string) string {
	fmt.Print(msg)
	scanner := bufio.NewScanner(os.Stdin)
	scanner.Scan()
	return strings.TrimSpace(scanner.Text())
}

func main() {
	_ = godotenv.Load()

	apiID, _ := strconv.Atoi(os.Getenv("TG_API_ID"))
	apiHash := os.Getenv("TG_API_HASH")
	phone := os.Getenv("TG_PHONE")

	logger, _ := zap.NewDevelopment()
	defer logger.Sync()

	ctx := context.Background()

	client := telegram.NewClient(apiID, apiHash, telegram.Options{
		SessionStorage: &session.FileStorage{Path: "session.json"},
		Logger:         logger,
		Device: telegram.DeviceConfig{
			DeviceModel:    "MacBook Pro",
			SystemVersion:  "macOS 14.0",
			AppVersion:     "1.0.0",
			SystemLangCode: "uk",
			LangCode:       "uk",
		},
	})

	if err := client.Run(ctx, func(ctx context.Context) error {
		// --- 1. Auth ---
		flow := auth.NewFlow(
			auth.CodeOnly(phone, auth.CodeAuthenticatorFunc(func(ctx context.Context, sentCode *tg.AuthSentCode) (string, error) {
				fmt.Println("\n--- Auth ---")
				return scanString("Enter code from Telegram: "), nil
			})),
			auth.SendCodeOptions{},
		)

		if err := client.Auth().IfNecessary(ctx, flow); err != nil {
			if strings.Contains(err.Error(), "SESSION_PASSWORD_NEEDED") {
				pwd := scanString("Enter 2FA password: ")
				_, err := client.Auth().Password(ctx, pwd)
				if err != nil {
					return err
				}
			} else {
				return err
			}
		}

		fmt.Println("Login succeed!")
		api := client.API()

		// --- 2. Get list of chats ---
		fmt.Println("\nLoading list of chats...")
		dialogs, err := api.MessagesGetDialogs(ctx, &tg.MessagesGetDialogsRequest{
			Limit:      20,
			OffsetPeer: &tg.InputPeerEmpty{},
		})
		if err != nil {
			return err
		}

		modified, ok := dialogs.AsModified()
		if !ok {
			return fmt.Errorf("can't list chats")
		}

		usersMap := make(map[int64]*tg.User)
		for _, u := range modified.GetUsers() {
			if user, ok := u.AsNotEmpty(); ok {
				usersMap[user.ID] = user
				}
		}

		type ChatEntry struct {
			Title string
			Peer  tg.InputPeerClass
		}
		var myChats []ChatEntry

		fmt.Println("\n--- Your last chats ---")
		for _, dlg := range modified.GetDialogs() {
			p := dlg.GetPeer()
			entry := ChatEntry{}

			switch peer := p.(type) {
			case *tg.PeerUser:
				if u, ok := usersMap[peer.UserID]; ok {
					entry.Title = fmt.Sprintf("%s %s", u.FirstName, u.LastName)
					entry.Peer = &tg.InputPeerUser{UserID: u.ID, AccessHash: u.AccessHash}
				}
			case *tg.PeerChat:
				for _, c := range modified.GetChats() {
					if ch, ok := c.AsNotEmpty(); ok && ch.GetID() == peer.ChatID {
						entry.Title = ch.GetTitle()
						entry.Peer = &tg.InputPeerChat{ChatID: ch.GetID()}
					}
				}
			case *tg.PeerChannel:
				for _, c := range modified.GetChats() {
					if ch, ok := c.(*tg.Channel); ok && ch.ID == peer.ChannelID {
						entry.Title = ch.Title
						entry.Peer = &tg.InputPeerChannel{ChannelID: ch.ID, AccessHash: ch.AccessHash}
					}
				}
			}

			if entry.Peer != nil {
				fmt.Printf("[%d] %s\n", len(myChats), entry.Title)
				myChats = append(myChats, entry)
			}
		}

		// --- 3. List chats and select chat ---
		if len(myChats) == 0 {
			fmt.Println("No chats.")
			return nil
		}

		choiceStr := scanString("\nSelect chat number: ")
		idx, _ := strconv.Atoi(choiceStr)

		if idx < 0 || idx >= len(myChats) {
			return fmt.Errorf("wrong chat")
		}
		selected := myChats[idx]

		history, err := api.MessagesGetHistory(ctx, &tg.MessagesGetHistoryRequest{
			Peer:  selected.Peer,
			Limit: 10,
		})
		if err != nil {
			return err
		}

		fmt.Printf("\n--- History: %s ---\n", selected.Title)
		if h, ok := history.AsModified(); ok {
			msgs := h.GetMessages()
			for i := len(msgs) - 1; i >= 0; i-- {
				if m, ok := msgs[i].(*tg.Message); ok {
					sender := "He/She"
					if m.Out {
						sender = "You"
					}
					fmt.Printf("[%s]: %s\n", sender, m.Message)
				}
			}
		}

		// --- 4. Write a message ---
		text := scanString(fmt.Sprintf("\nMessage for %s: ", selected.Title))
		if text != "" {
			_, err = api.MessagesSendMessage(ctx, &tg.MessagesSendMessageRequest{
				Peer:     selected.Peer,
				Message:  text,
				RandomID: time.Now().UnixNano(),
			})
			if err != nil {
				return err
			}
			fmt.Println("🚀 Sent!")
		}

		return nil
	}); err != nil {
		log.Fatal("Client error: ", err)
	}
}
```
