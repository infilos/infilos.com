## Work In Progress

<div align="center">
<img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/tumblr_o68i2aVvlE1s9f4joo1_500.gif" style="display:block;width:70%;" alt="NAME" align=center />
</div>

## Font Color

```html
<font color=red>red</font>
<font color=blue>blue</font>
<font color=green>green</font>
```

## Alert

```
{{% alert title="Warning" color="warning" %}}
This is a warning.
{{% /alert %}}
```

- primary
- info
- warning

## Page Info

```
{{% pageinfo color="primary" %}}
This is placeholder content.
{{% /pageinfo %}}
```

## Tabbed Panes

```
{{< tabpane >}}
  {{< tab header="English" >}}
    Welcome!
  {{< /tab >}}
  {{< tab header="German" >}}
    Herzlich willkommen!
  {{< /tab >}}
  {{< tab header="Swahili" >}}
    Karibu sana!
  {{< /tab >}}
{{< /tabpane >}}
```

## Card

```
{{< card header="**Imagine**" title="Artist and songwriter: John Lennon" subtitle="Co-writer: Yoko Ono"
          footer="![SignatureJohnLennon](https://server.tld/…/signature.png \"Signature John Lennon\")">>}}
Imagine there's no heaven, It's easy if you try<br/>
No hell below us, above us only sky<br/>
Imagine all the people living for today…

…
{{< /card >}}
```

## Card Code

```
{{< card-code header="**C**" lang="C" >}}
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
  puts("Hello World!");
  return EXIT_SUCCESS;
}
{{< /card-code >}}
```

## Card Group

```
{{< cardpane >}}
  {{< card header="Header card 1" >}}
    Content card 1
  {{< /card >}}
  {{< card header="Header card 2" >}}
    Content card 2
  {{< /card >}}
  {{< card header="Header card 3" >}}
    Content card 3
  {{< /card >}}
{{< /cardpane >}}
```
