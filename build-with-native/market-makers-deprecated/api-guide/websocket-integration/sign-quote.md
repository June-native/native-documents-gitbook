# Sign-Quote

### Receiving Sign Quote request

After our server has validated the firm quote, we will request you to sign the quote with the following message:

```
{
  "messageType": "signQuote",
  "message": {
    "chainId": number,
    "quoteId": number,
    // <OPTIONAL> In wei, the amount that the market maker is allowed to borrow from native's credit pool
    "availableBorrowBalance": string,
    "isBorrowing": boolean, <OPTIONAL> boolean value to indicate whether you want to use native's credit pool
    "quoteData": {
      // It is leveraged to mitigate quote replay.
      "id": number,
      "signer": string,
      "buyer": string,
      "seller": string,
      "buyerToken": string,
      "sellerToken": string,
      "buyerTokenAmount":  string,
      "sellerTokenAmount": string,
      "deadlineTimestamp": string,
      "caller": string,
      "quoteId": string,
    }
  }
}
```

### Sending Sign Quote response

You need to sign the quote based on the EIP712 specification with our order format. Refer to this guide for an example of signing transactions. After signing, send the response back to Native in the following format:

```
{
  "messageType": "signature",
  "message": {
    "quoteId": string,  // This is the quote id previously sent.
    "signature": string  // 0x-prefix signature (65 bytes)
  }
}
```

{% hint style="info" %}
**Note**: The Native server will generally wait **up to 350 milliseconds** for a quote to be signed before the quote is flagged as invalid. Too many signature failures can result in an order pause for the Market Maker, as it affects the user experience for those trading through Native.
{% endhint %}

{% hint style="success" %}
Congratulations! You have successfully integrated with Native and are now a Market Maker.
{% endhint %}
