# rest

`rest` is a Rust crate making it easier to write clients for REST APIs.
If you find yourself building lots of wrappers around highly structured APIs, odds are good `rest` can help.
It lets you write your boilerplate once, and comes with prewritten boilerplate for common API designs.

The key assumptions that `rest` makes, which your API needs to follow, are:
- Everything's based around resources.
  (They might not be called that, but that's what `rest` calls them.)
- Those resources have at least one of the usual REST operations: List all, get one, create, edit, and delete.
- The way you perform operations is reasonably consistent across all the resources

The defaults all assume a very typical REST API, and should "just work" with minimal customization for most of them.
But `rest` leaves lots of room for customization.
For example, it's common for login endpoints to not really fit the "resource" model, and the intended way to implement them is with simple methods -- which can still call back into your REST stack where it makes sense.

## Example

Here's a quick snippet, implementing a terrible client for a terrible chat server:

```rs
// please remember that rest is extremely early on. this code might not work
// yet or the api might have changed without this being updated yet.

// TODO: What's the best way to parameterize `Api`'s autoimplementation?
// Currently, the contained types are assumed to implement... some trait? so
// they can act like middleware that converts between objects and HTTP.
#[rest::api]
struct ChatApi(rest::url::Base, rest::auth::Basic, rest::body::Json, rest::paging::InBody);
impl ChatApi {
  pub async fn new(username: &str) -> rest::Result<Self> {
    Ok(Self(
      rest::url::Base("https://chat.example.com/v2"),
      rest::auth::Basic(username, ""),
      rest::body::Json,
      rest::paging::InBody("data", "page", "pages"),
    ))
  }
}

#[rest::endpoint(ChatApi, "/channels", operations(list, get, create))]
struct Channel {
  #[field(id)]
  name: String,
  #[field(serverside)]
  created: chrono::DateTime,
}

#[rest::endpoint(Channel, "/messages", operations(list, get, create, patch))]
struct Message {
  #[field(id, serverside)]
  id: usize,
  #[field(serverside)]
  sent: chrono::DateTime,
  author: String,
  contents: String,
}

#[tokio::main]
async fn main() -> rest::Result<()> {
  let mut args = std::env::args().skip(1);

  let username = args.next().expect("must provide a username");
  let chatserver = ChatApi::new(&username).await?;
  match args.next().expect("must provide a verb").as_str() {
    "history" => {
      let chan = args.next().expect("must provide channel name");
      let channel = chatserver.get::<Channel>(&chan).await?;
      println!("#{} ({})", channel.name, channel.created.to_rfc2822());
      let messages = channel.list::<Message>();
      while let Some(msg) = messages.next().await? {
        println!("{} {}: {}", msg.sent.to_rfc2822(), msg.author, msg.contents);
      }
    }
    "open" => {
      let name = args.next().expect("must provide channel name");
      chatserver.create::<Channel>().name(name).await?;
    }
    "send" => {
      let chan = args.next().expect("must provide channel name");
      let author = args.next().expect("msut provide username");
      let contents = args.next().expect("must provide message contents");
      // TODO: Is 'overloading' `.get. like this to refer to things good?
      // Or would it be better to have a separate method, that isn't a verb?
      let new = chatserver.get::<Channel>(&chan).create::<Message>().author(author).contents(contents).await?;
      println!("Sent. ID: {}; time: {}", new.id, new.sent.to_rfc2822());
    }
    "edit" => {
      let chan = args.next().expect("must provide channel name");
      let id = args.next().map(|s| s.parse()).expect("must provide message ID");
      let contents = args.next().expect("must provide message contents");
      chatserver.get::<Channel>(&chan).patch::<Message>(id).contents(contents)
    }
    _ => ()
  }
  Ok(())
}
```
