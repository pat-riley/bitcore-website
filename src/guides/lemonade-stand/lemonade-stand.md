# Lemonade Stand

This tutorial will go over how to accept bitcoin payments via a web site directly to your wallet. We will interact with a Bitcore node from a web browser to receive payment notifications. And we will generate new keys on the server side for each purchase.

## Create a Service

Please refer to the [service development document](service-development.html).

## Generate Invoice Page

For our simple application, we're going to have two pages. One page to generate an invoice, and another page to display and pay the invoice.

Let's start with generating an invoice. We'll need a route:

```js
LemonadeStand.prototype.setupRoutes = function(app, express) {
  app.use('/', express.static(__dirname + '/static'));
};

LemonadeStand.prototype.getRoutePrefix = function() {
  return 'payments';
};
```

Create a `payments/static` and put an `index.html` in it:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Lemonade Stand</title>
</head>

  <body>
    <h1>Lemonade Stand</h1>
    <h2>Invoice</h2>

    <form method="post" action="/payments/invoice">
      Amount: <input type="text" name="amount"/> BTC <input type="submit" value="Generate Invoice" />
    </form>
  </body>
</html>
```

You can access this page at `http://localhost:3001/payments`.

## Pay Invoice Page

Next we'll need to be able to generate the invoice server-side and display a page to pay the invoice. Back to `payments/index.js`:

```js
var inherits = require('util').inherits;
var EventEmitter = require('events').EventEmitter;
var fs = require('fs');
var bitcore = require('bitcore-lib');
var bodyParser = require('body-parser');

function LemonadeStand(options) {
  EventEmitter.call(this);

  this.node = options.node;

  this.invoiceHtml = fs.readFileSync(__dirname + '/invoice.html', 'utf8');

  // Use 1 HD Private Key and generate a unique address for every invoice
  this.hdPrivateKey = new bitcore.HDPrivateKey(this.node.network);
  this.addressIndex = 0;
}

inherits(LemonadeStand, EventEmitter);

LemonadeStand.dependencies = ['bitcoind', 'db', 'address'];

LemonadeStand.prototype.start = function(callback) {
  setImmediate(callback);
};

LemonadeStand.prototype.stop = function(callback) {
  setImmediate(callback);
};

LemonadeStand.prototype.getAPIMethods = function() {
  return [];
};

LemonadeStand.prototype.getPublishEvents = function() {
  return [];
};

LemonadeStand.prototype.setupRoutes = function(app, express) {
  var self = this;

  app.use(bodyParser.urlencoded({extended: true}));

  app.use('/', express.static(__dirname + '/static'));

  app.post('/invoice', function(req, res, next) {
    self.addressIndex++;
    self.amount = parseFloat(req.body.amount) * 1e8;
    res.status(200).send(self.filterInvoiceHTML());
  });
};

LemonadeStand.prototype.getRoutePrefix = function() {
  return 'payments';
};

LemonadeStand.prototype.filterInvoiceHTML = function() {
  var btc = this.amount / 1e8;
  var address = this.hdPrivateKey.derive(this.addressIndex).privateKey.toAddress();
  var transformed = this.invoiceHtml
    .replace(/{{amount}}/g, btc)
    .replace(/{{address}}/g, address)
  return transformed;
};

module.exports = LemonadeStand;
```

Let's walk through this. In our constructor we are loading the html data into memory. We'll use this later to replace variables. In a normal application, you would probably use a templating engine like Handlebars or Mustache.

We generate a random HDPrivateKey from which all of our invoice addresses will be generated. Normally what you would do is generate this on a separate computer and put the corresponding HDPublicKey on the server. That way if your server is hacked, your bitcoin is still safe!

Each new invoice bumps the addressIndex by 1. This will create a unique bitcoin address for every invoice that is generated.

In `setupRoutes()` we add an express middleware for parsing form data. Then we add a handler for `/invoice`. When the user submits the form from the first page we created, `amount` is passed to this `/invoice` route. We then parse the amount, and respond back with the html, replacing our placeholder values with real values.

Here is what `payments/invoice.html` looks like:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Lemonade Stand</title>
  <script src="/socket.io/socket.io.js"></script>
  <script src="https://code.jquery.com/jquery-2.1.4.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery.qrcode/1.0/jquery.qrcode.min.js"></script>
  <script src="/payments/bitcore-lib.js"></script>
</head>

<body>
  <h1>Lemonade Stand</h1>
  <h2>Invoice</h2>
  <div id="qrcode"></div>
  <p>Please send {{amount}} BTC to {{address}}</p>
  <h2>Total Received</h2>
  <p><span id="total-{{address}}">0</span> BTC</p>

  <script type="text/javascript">
    $('#qrcode').qrcode("bitcoin:{{address}}?amount={{amount}}");
  </script>

  <script language="javascript">
    var bitcore = require('bitcore-lib');
    var socket = io('http://localhost:3001');
    socket.on('address/balance', function(addressObj, balance) {
      // The address object includes hash, type, and network. Use bitcore to derive the address.
      var address = bitcore.Address(addressObj);
      document.getElementById('total-' + address).innerHTML = (balance / 1e8);
    });
    socket.emit('subscribe', 'address/balance', ['{{address}}']);
  </script>
</body>

</html>
```

This will generate a QR code with the bitcoin address and amount. It subscribes to the `address/balance` event from the `address` service for the given address. Every time the balance changes, we receive an event over socket.io and we update the total received accordingly.

It uses bitcore on the client side to derive the address from the address object. You will want to download `bitcore-lib.js` and put it in your `payments/static` directory.

## Conclusion

So there we've built our very own bitcoin accepting lemonade stand using Bitcore Node!
