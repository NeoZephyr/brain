```go
type Player struct {
    Name string `json:"name"`
    Age int32 `json:"age"`
}

player := Player{
    Name: "jack",
}

// bytes, _ := json.Marshal(&player)
bytes, _ := json.Marshal(player)
fmt.Println(string(bytes))
```

```go
playerJson := "{\"name\":\"jack\",\"age\":0}"

var p interface{}
err := json.Unmarshal([]byte(playerJson), &p)

// m, ok := p.(map[string]interface{})
// fmt.Println(m, ok)

switch v := p.(type) {
case Player:
    fmt.Println("player", v.Name, v.Age)
case map[string]interface{}:
    fmt.Println("map[string]interface{}", v["name"], v["age"])
default:
    fmt.Println("default")
}
```