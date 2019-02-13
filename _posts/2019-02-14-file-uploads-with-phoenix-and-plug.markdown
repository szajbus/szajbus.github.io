---
layout: "post"
title: "Base64-encoded file uploads with Phoenix and Plug"
date: "2019-02-13 09:00"
categories: Elixir
excerpt: "Base64-encoded files may be a special case in your app, but with the help of Plug.Upload they can be handled as easily as regular files uploaded via HTML forms."
cover: "/assets/images/plug.jpg"
cover_credits: >
  Photo by <a href="https://unsplash.com/photos/ZUabNmumOcA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText" rel="nofollow">Steve Johnson</a> on Unsplash
---

Handling external files is one of the basic tasks for many web applications. Profile pictures, attachments, resources fetched from remote services, all are examples of the above.

The most common way such files find their way into a web app is by means of an HTTP multipart request, usually performed through a web page with an HTML form and a `file` input tag or a client using app's HTTP API directly. Phoenix and, more specifically, Plug handles those very well, by wrapping the uploaded file information in a convenient struct:

```elixir
%Plug.Upload{
  content_type: "image/jpg",
  filename: "rainbow.jpg",
  path: "/var/folders/nk/_hgjvryd2ml40g0jpyllt2080000gn/T//plug-1548/multipart-440190-911831-1"
}
```

Plug writes uploaded data on disk as a temporary file and provides its path along with original filename and type encapsulated in a struct. As such, it is passed to Phoenix controller with the rest of request params.

The convenience of `Plug.Upload` goes beyond the definition of the above struct. It also implements a `GenServer` that tracks and manages all file uploads during the life cycle of the process [^1]. This includes generation of unique file names, creation of intermediate directories in _tmp_ folder, creating and writing the file itself and, probably most importantly, cleanup of all the temporary files when the process terminates.

However, not all the clients will find the HTTP multipart interface suitable. Instead of reimplementing mentioned `Plug.Upload` functionality, it would be better to simply re-use it when handling files obtained by other means.

Let's explore some options.

## Base64-encoded data

XML and JSON are just two examples of text-based formats widely used in service-to-service communication. They are easy to implement and very flexible, but are limited to text only. Neither of those formats, nor the protocols that make use of them, directly support mixing with non-text data.

The solution is to turn non-text (binary) data into plain text, that can be easily embedded in XML or JSON for transport and turned back into binary format at the destination. Base64 is probably the most common encoding scheme that helps to achieve this.

Consider the following JSON document, describing some person and embedding her picture as Base64-encoded string (truncated for readability):

```json
{
  "name": "John Doe",
  "email": "john@example.org",
  "picture": "SGFuZGxpbmcgZXzEtMSIKfQpgYGA..."
}
```

In contrast to HTTP multipart requests, this time Phoenix controller will receive `picture` as string instead of `%Plug.Upload{}` struct. To plug into the above-mentioned conveniences of `Plug.Upload` we need to decode it and build the struct ourselves.

First, let's define a helper function that writes arbitrary binary data into a temporary file and wraps it with a `%Plug.Upload{}` struct [^2].

```elixir
def binary_to_upload(binary) do
  with {:ok, path} <- Plug.Upload.random_file("upload"),
       {:ok, file} <- File.open(path, [:write, :binary]),
       :ok <- IO.binwrite(file, binary),
       :ok <- File.close(file) do
    %Plug.Upload{path: path}
  end
end
```

The important part is the call to `Plug.Upload.random_file/1`, which creates a file in _tmp_ directory and sets up its tracking by the `GenServer` so that it can be later cleaned up.

Now the handling of Base64-encoded files is straightforward.

```elixir
def base64_to_upload(str) do
  with {:ok, data} <- Base.decode64(str) do
    binary_to_upload(data)
  end
end
```

Base64 encoding is quite common, but it's not the only alternative. Let's see another one.

## Data URLs

Mobile and JavaScript single page apps, even when using regular HTTP APIs, may prefer to upload files as [Data URLs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs) that include raw or Base64-encoded file contents.

The format is very simple, it consists of `data:` prefix, optional MIME type indicator, optional `base64` token (for non-textual data) and the data itself. For example, `data:text/plain;base64,SGVsbG8gd29ybGQh` is a `"Hello World!"` string encoded as Data URL. You can try pasting it in your web browser's address bar and it should happily decode it.

The approach here is analogous to the previous example - our goal is to decode the original file and plug it into `Plug.Upload`.

The following code snippet uses [ex_url](https://hex.pm/packages/ex_url) package, which provides parser and decoder for Data URLs (note `URL.Data.parse/1` function).

```elixir
def data_url_to_upload(data_url) do
  with %{scheme: "data"} = uri <- URI.parse(data_url),
       %URL.Data{data: data} <- URL.Data.parse(uri) do
    binary_to_upload(data)
  end
end
```

Let's see it in action:

```elixir
iex> upload = data_url_to_upload("data:text/plain;base64,SGVsbG8gd29ybGQh")
%Plug.Upload{
  content_type: nil,
  filename: nil,
  path: "/var/folders/nk/_hgjvryd2ml40g0jpyllt2080000gn/T//plug-1548/base64-data-1548887956-746460942629208-8"
}

iex> File.read(upload.path)
{:ok, "Hello World!"}
```

Looks good!

## Summary

> The essence of abstractions is preserving information that is relevant in a given context, and forgetting information that is irrelevant in that context.
>
> – John V. Guttag [^3]

Base64-encoded files may be a special case in your app, but with little work they can be handled as easily as regular files uploaded via HTML forms.

Plug.Upload nicely abstracts away the process of handling temporary files. At the same time, it provides a convenient struct around them to work with.

With it at your disposal, irrespective of where the external files come from, what format they're in and what your application does with them, you can rely on a universal, stable interface to ease both feature development and testing.

[^1]: Wherever in this article a word _process_ is used, it refers to Erlang VM process.

[^2]: To keep the examples simple, I am not even trying to set `content_type` and `filename` fields in the structs. Depending on the situation they may or may not be available in the request data.

[^3]: Guttag, John V. (2013-01-18). _Introduction to Computation and Programming Using Python_ (Spring 2013 ed.). Cambridge, Massachusetts: The MIT Press. ISBN 9780262519632.
