# BitSPV Pay Integration Guide

A concise guide for integrating Bitcoin SV micropayments using BitSPV Pay service.

## Overview

BitSPV Pay provides a simple payment gateway for Bitcoin SV micropayments with OP_RETURN data verification. Perfect for paywalls, content monetization, and micro-transactions.

## Quick Integration

### 1. Basic Setup

```javascript
const PAY_BASE_URL = 'https://pay.bitspv.com';
const CALLBACK_URL = window.location.href.split('?')[0];
```

### 2. Payment Parameters

```javascript
const paymentParams = [
    {
        data: [hexEncodedData], // OP_RETURN data in hex
        satoshis: 0,            // OP_RETURN output (0 satoshis)
    },
    {
        address: 'your_bsv_address', // Your receiving address
        satoshis: 1,                 // Payment amount in satoshis
    },
];
```

### 3. Initiate Payment

```javascript
function initiatePayment() {
    const orderId = generateUUID();
    const bData = [`YOUR_APP_PAYMENT_${orderId}`];
    
    const paymentParams = [
        {
            data: bData.map(d => stringToHex(d)),
            satoshis: 0,
        },
        {
            address: 'your_address_here',
            satoshis: 1,
        },
    ];
    
    const encodedData = encodeURIComponent(JSON.stringify(paymentParams));
    const callbackUrl = encodeURIComponent(CALLBACK_URL);
    
    // Popup mode
    window.open(
        `${PAY_BASE_URL}/?mode=popup&callbackUrl=${callbackUrl}#${encodedData}`,
        '_blank',
        'width=500,height=580,scrollbars=yes,resizable=yes'
    );
    
    // Redirect mode (fallback)
    // window.location.href = `${PAY_BASE_URL}/?mode=redirect&callbackUrl=${callbackUrl}#${encodedData}`;
}
```

### 4. Handle Payment Response

#### PostMessage (Popup Mode)
```javascript
window.addEventListener('message', (event) => {
    if (event.origin !== PAY_BASE_URL) return;
    
    const messageData = typeof event.data === 'string' ? 
        JSON.parse(event.data) : event.data;
    
    switch (messageData.type) {
        case 'payment_success':
            verifyPayment(messageData.payload.txid);
            break;
        case 'payment_error':
            handlePaymentError(messageData.error);
            break;
        case 'payment_cancelled':
            handlePaymentCancellation();
            break;
    }
});
```

#### URL Callback
```javascript
function checkPaymentCallback() {
    const urlParams = new URLSearchParams(window.location.search);
    const txid = urlParams.get('data');
    const status = urlParams.get('status');
    
    if (status === 'error') {
        // Handle payment error
        return;
    }
    
    if (txid) {
        verifyPayment(txid);
    }
}
```

### 5. Payment Verification

```javascript
async function verifyPayment(txid) {
    try {
        // Fetch transaction details from blockchain
        const response = await fetch(
            `https://api.whatsonchain.com/v1/bsv/main/tx/hash/${txid}`
        );
        const txData = await response.json();
        
        // Extract OP_RETURN data
        const opReturns = txData.vout
            .filter(output => output.value === 0 && 
                    output.scriptPubKey?.type === 'nulldata')
            .map(output => output.scriptPubKey.asm.replace('OP_RETURN ', ''));
        
        // Verify OP_RETURN contains expected data
        const expectedData = stringToHex(`YOUR_APP_PAYMENT_${orderId}`);
        const opReturnData = opReturns.join('');
        
        if (opReturnData.includes(expectedData)) {
            // Payment verified - unlock content
            unlockPaidContent();
        } else {
            throw new Error('Payment verification failed');
        }
        
    } catch (error) {
        console.error('Verification failed:', error);
    }
}
```

## Integration Modes

### Popup Mode
- Opens payment in new window
- Better UX, stays on your site
- Requires popup handling
- Uses PostMessage for communication

### Redirect Mode
- Redirects to payment page
- Fallback for blocked popups
- Returns via callback URL
- Simpler implementation

## Security Considerations

1. **Always verify payments** on blockchain
2. **Validate OP_RETURN data** matches expected format
3. **Check payment expiry** if implementing time-based access
4. **Sanitize callback URLs** to prevent XSS
5. **Use HTTPS** for all communications

## Error Handling

- Handle popup blocking gracefully
- Implement fallback to redirect mode
- Validate transaction data thoroughly
- Provide clear user feedback
- Cache verified payments locally

## Example Use Cases

- **Paywalls**: Unlock premium content
- **Micro-donations**: Support creators
- **API Access**: Pay-per-use services
- **Digital Downloads**: Instant access after payment
- **Subscription Gates**: Time-based access control

## Browser Compatibility

- Modern browsers with ES6+ support
- Popup blocking may require user interaction
- PostMessage API for cross-window communication
- LocalStorage for payment caching
