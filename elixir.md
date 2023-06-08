# Elixir

## Reduce vs Fold

[Source](https://github.com/tacticiankerala/elixir-weather/blob/25e186c3808396ad03d0ab34b3bf24e918e20aab/lib/weather/ascii.ex#L8C1-L16)

~~~elixir
def convert_to_ascii_art(list) do
  [head | tail] = list
                  |> Enum.map(&ascii_char/1)
                  |> Enum.map(&String.split(&1, "\n"))

  tail
  |> Enum.reduce(head, &join_char/2)
  |> Enum.join("\n")
end
~~~

In order to use `Enum.reduce/3` we have to provide accumulator - an initial element for reducing. That is why we destruct `list` by its `head`, which would be accumulator, and its `tail`, which we provide as first argument to `Enum.reduce/3`.

It turns out, in Elixir we have overload for `Enum.reduce`, which is [`Enum.reduce/2`](https://hexdocs.pm/elixir/1.12.3/Enum.html#reduce/2). This version doesn't require an initial element, so we can rewrite our code, continuing pipe throughout the whole function.

~~~elixir
def convert_to_ascii_art(list) do
  list
  |> Enum.map(&ascii_char/1)
  |> Enum.map(&String.split(&1, "\n"))
  |> Enum.reduce(&join_char/2)
  |> Enum.join("\n")
end
~~~
