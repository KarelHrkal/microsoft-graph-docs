---
description: "Automatically generated file. DO NOT MODIFY"
---

```javascript

const options = {
	authProvider,
};

const client = Client.init(options);

const place = {
  @odata.type: "microsoft.graph.room",
  nickname: "Conf Room",
  building: "1",
  label: "100",
  capacity: "50",
  isWheelchairAccessible: false
};

let res = await client.api('/places/cf100@contoso.com')
	.version('beta')
	.update(place);

```