# Django-Stripe
put stripe payment in your django project

1- create your django project

2- install stripe package

<code> pip install stripe </code>

3- add stripe <code>STRIPE_PUBLISHABLE_KEY</code> and <code>STRIPE_SECRET_KEY</code>
in <code>yourproject/settings.py</code>

for test only
```python
STRIPE_PUBLISHABLE_KEY = 'pk_test_51K7Gw5La4Cx0tSz0pqJCzSfc4CgPl4QnACurOKqAomKrpFUWSMGIx9khGI5mkjmW3BglFhvfWdUPjlt6HaOr2kyZ0067oCOPEN'
STRIPE_SECRET_KEY = 'sk_test_51K7Gw5La4Cx0tSz0DDwVdFdv8HGffxc8MKU7e2iofGAee6Uyft3NmtDNmexjZpScGGc0jVMUw7TkROd4az2KgzO300BqLZKh8b'
```
and in your views.py:
(for test only)
```python
stripe.api_key = "sk_test_51K7Gw5La4Cx0tSz0DDwVdFdv8HGffxc8MKU7e2iofGAee6Uyft3NmtDNmexjZpScGGc0jVMUw7TkROd4az2KgzO300BqLZKh8b"
```

# you need to make product,order_item/cart,etc models
4- now lets create <code>models</code>

now in <code>yourapp/models.py</code> add this code:

order model:
```python
class Order(models.Model):
  
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    items = models.ManyToManyField(OrderItem)
    start_date = models.DateTimeField(auto_now_add=True,null=True)
    orderd_date = models.DateTimeField(null=True)
    ordered = models.BooleanField(default=False)
    billing_address = models.ForeignKey('BillingAddress', on_delete=models.SET_NULL, blank=True, null=True)
    payment = models.ForeignKey('Payment', on_delete=models.SET_NULL, blank=True, null=True)
    
    def get_total(self):
        total = 0
        for order_item in self.items.all():
            total += order_item.get_final_price()
        return total
```

payment model:
```python
class Payment(models.Model):</code>

    stripe_charge_id = models.CharField(max_length=100,null=True)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL,blank=True, null=True)
    amount = models.FloatField()
    timestamp = models.DateTimeField(auto_now_add=True)
 ```
5- now in <code>yourapp/views.py</code>

## dont forget import your models EXAMPLE: FROM .MODELS IMPORT ALL 
## and you need to (import stirpe) after install it 
## from django.views.generic import ListView , DetailView, View

first we need to create <code>PaymentView</code>

```python
class PaymentView(View):
  
    def get(self,*args,**kwargs):
        order = Order.objects.get(user=self.request.user, ordered=False)
        context = {
            'order':order
        }
        return render(self.request,'YOUR TEMPLATE NAME',context)</code>
```
now its turn POST function :

```python
  def post(self,*args,**kwargs):
  
        order = Order.objects.get(user=self.request.user, ordered=False)
        token = self.request.POST.get('stripeToken')
        amount = int(order.get_total() * 100) 
```

in post function lets add Payemnt and charge:

```python
            charge = stripe.Charge.create(
            amount= amount,
            currency="usd",
            description="My second Test Charge (created for API docs)",
            source= token, # obtained with Stripe.js
            )
            payment = Payment()
            payment.stripe_charge_id = charge['id']
            payment.user = self.request.user
            payment.amount = order.get_total()
            payment.save()


            order_items = order.items.all()
            order_items.update(ordered=True)

            for item in order_items:
                item.save()

            order.ordered = True
            order.payment = payment
            order.save()
  
            ## for messages (from django.contrib import messages)
            messages.success(self.request,'Your order was successful')
            return redirect('/')
```

okay lets start with the hard thig

the error handling :

firts step put the payment and charge function in (try) like this:

```python
    try:
            charge = stripe.Charge.create(
            amount= amount,
            currency="usd",
            description="My second Test Charge (created for API docs)",
            source= token, # obtained with Stripe.js
            )
            payment = Payment()
            payment.stripe_charge_id = charge['id']
            payment.user = self.request.user
            payment.amount = order.get_total()
            payment.save()


            order_items = order.items.all()
            order_items.update(ordered=True)

            for item in order_items:
                item.save()

            order.ordered = True
            order.payment = payment
            order.save()

            messages.success(self.request,'Your order was successful')
            return redirect('/')
```

## now this code form https://stripe.com/docs/error-handling

## its for avoid and errors

```python

        except stripe.error.CardError as e:
            body = e.json_body
            err = body.get('error', {})
            messages.warning(self.request, f"{err.get('message')}")
            return redirect("/")

        except stripe.error.RateLimitError as e:
                # Too many requests made to the API too quickly
            messages.warning(self.request, "Rate limit error")
            return redirect("/")

        except stripe.error.InvalidRequestError as e:
                # Invalid parameters were supplied to Stripe's API
            print(e)
            messages.warning(self.request, "Invalid parameters")
            return redirect("/")

        except stripe.error.AuthenticationError as e:
                # Authentication with Stripe's API failed
                # (maybe you changed API keys recently)
            messages.warning(self.request, "Not authenticated")
            return redirect("/")

        except stripe.error.APIConnectionError as e:
                # Network communication with Stripe failed
            messages.warning(self.request, "Network error")
            return redirect("/")

        except stripe.error.StripeError as e:
                # Display a very generic error to the user, and maybe send
                # yourself an email
            messages.warning(
                    self.request, "Something went wrong. You were not charged. Please try again.")
            return redirect("/")

        except Exception as e:
                # send an email to ourselves
            messages.warning(
                self.request, "A serious error occurred. We have been notifed.")
            return redirect("/")


```


the final code in views.py will be like this:


```python
class PaymentView(View):

    def get(self,*args,**kwargs):
        order = Order.objects.get(user=self.request.user, ordered=False)
        context = {
            'order':order
        }
        return render(self.request,'pages/payment.html',context)

    def post(self,*args,**kwargs):
        order = Order.objects.get(user=self.request.user, ordered=False)
        token = self.request.POST.get('stripeToken')
        amount = int(order.get_total() * 100)

        try:
            charge = stripe.Charge.create(
            amount= amount,
            currency="usd",
            description="My second Test Charge (created for API docs)",
            source= token, # obtained with Stripe.js
            )
            payment = Payment()
            payment.stripe_charge_id = charge['id']
            payment.user = self.request.user
            payment.amount = order.get_total()
            payment.save()


            order_items = order.items.all()
            order_items.update(ordered=True)

            for item in order_items:
                item.save()

            order.ordered = True
            order.payment = payment
            order.save()

            messages.success(self.request,'Your order was successful')
            return redirect('/')
        except stripe.error.CardError as e:
            body = e.json_body
            err = body.get('error', {})
            messages.warning(self.request, f"{err.get('message')}")
            return redirect("/")

        except stripe.error.RateLimitError as e:
                # Too many requests made to the API too quickly
            messages.warning(self.request, "Rate limit error")
            return redirect("/")

        except stripe.error.InvalidRequestError as e:
                # Invalid parameters were supplied to Stripe's API
            print(e)
            messages.warning(self.request, "Invalid parameters")
            return redirect("/")

        except stripe.error.AuthenticationError as e:
                # Authentication with Stripe's API failed
                # (maybe you changed API keys recently)
            messages.warning(self.request, "Not authenticated")
            return redirect("/")

        except stripe.error.APIConnectionError as e:
                # Network communication with Stripe failed
            messages.warning(self.request, "Network error")
            return redirect("/")

        except stripe.error.StripeError as e:
                # Display a very generic error to the user, and maybe send
                # yourself an email
            messages.warning(
                    self.request, "Something went wrong. You were not charged. Please try again.")
            return redirect("/")

        except Exception as e:
                # send an email to ourselves
            messages.warning(
                self.request, "A serious error occurred. We have been notifed.")
            return redirect("/")
```

# its an example code 
# foucs in the id and classes its very important
the html code:

```html
  payment.html
  
  <main >
    <div class="container wow fadeIn">

      <h2 class="my-5 h2 text-center">Payment</h2>

      <div class="row">

        <div class="col-md-12 mb-4">
          <div class="card">
            <div class="current-card-form">
              <form action="." method="post" class="stripe-form">
                  {% csrf_token %}
                  <input type="hidden" name="use_default" value="true">
                  <div class="stripe-form-row">
                    <button id="stripeBtn">Submit Payment</button>
                  </div>
                  <div id="card-errors" role="alert"></div>
              </form>
            </div>

            <div class="new-card-form">
              <form action="." method="post" class="stripe-form" id="stripe-form">
                  {% csrf_token %}
                  <div class="stripe-form-row" id="creditCard">
                      <label for="card-element" id="stripeBtnLabel">
                          Credit or debit card
                      </label>
                      <div id="card-element" class="StripeElement StripeElement--empty"><div class="__PrivateStripeElement" style="margin: 0px !important; padding: 0px !important; border: none !important; display: block !important; background: transparent !important; position: relative !important; opacity: 1 !important;"><iframe frameborder="0" allowtransparency="true" scrolling="no" name="__privateStripeFrame5" allowpaymentrequest="true" src="https://js.stripe.com/v3/elements-inner-card-19066928f2ed1ba3ffada645e45f5b50.html#style[base][color]=%2332325d&amp;style[base][fontFamily]=%22Helvetica+Neue%22%2C+Helvetica%2C+sans-serif&amp;style[base][fontSmoothing]=antialiased&amp;style[base][fontSize]=16px&amp;style[base][::placeholder][color]=%23aab7c4&amp;style[invalid][color]=%23fa755a&amp;style[invalid][iconColor]=%23fa755a&amp;componentName=card&amp;wait=false&amp;rtl=false&amp;keyMode=test&amp;origin=https%3A%2F%2Fstripe.com&amp;referrer=https%3A%2F%2Fstripe.com%2Fdocs%2Fstripe-js&amp;controllerId=__privateStripeController1" title="Secure payment input frame" style="border: none !important; margin: 0px !important; padding: 0px !important; width: 1px !important; min-width: 100% !important; overflow: hidden !important; display: block !important; height: 19.2px;"></iframe><input class="__PrivateStripeElement-input" aria-hidden="true" aria-label=" " autocomplete="false" maxlength="1" style="border: none !important; display: block !important; position: absolute !important; height: 1px !important; top: 0px !important; left: 0px !important; padding: 0px !important; margin: 0px !important; width: 100% !important; opacity: 0 !important; background: transparent !important; pointer-events: none !important; font-size: 16px !important;"></div></div>
                  </div>
                  <div class="stripe-form-row">
                    <button id="stripeBtn">Submit Payment</button>
                  </div>
                  <div class="stripe-form-row">
                    <div class="custom-control custom-checkbox">
                      <input type="checkbox" class="custom-control-input" name="save" id="save_card_info">
                      <label class="custom-control-label" for="save_card_info">Save for future purchases</label>
                    </div>
                  </div>
                  <div id="card-errors" role="alert"></div>
              </form>
            </div>
          </div>
        </div>
      </div>
    </div>

</main>
```

now the javascript code:

```javascript  
<script src="https://js.stripe.com/v3/"></script>
<script nonce="">  // Create a Stripe client.
## you should change this public key this one only for testing
  var stripe = Stripe('pk_test_51K7Gw5La4Cx0tSz0pqJCzSfc4CgPl4QnACurOKqAomKrpFUWSMGIx9khGI5mkjmW3BglFhvfWdUPjlt6HaOr2kyZ0067oCOPEN');

  // Create an instance of Elements.
  var elements = stripe.elements();

  // Custom styling can be passed to options when creating an Element.
  // (Note that this demo uses a wider set of styles than the guide below.)
  var style = {
    base: {
      color: '#32325d',
      fontFamily: '"Helvetica Neue", Helvetica, sans-serif',
      fontSmoothing: 'antialiased',
      fontSize: '16px',
      '::placeholder': {
        color: '#aab7c4'
      }
    },
    invalid: {
      color: '#fa755a',
      iconColor: '#fa755a'
    }
  };

  // Create an instance of the card Element.
  var card = elements.create('card', {style: style});

  // Add an instance of the card Element into the `card-element` <div>.
  card.mount('#card-element');

  // Handle real-time validation errors from the card Element.
  card.addEventListener('change', function(event) {
    var displayError = document.getElementById('card-errors');
    if (event.error) {
      displayError.textContent = event.error.message;
    } else {
      displayError.textContent = '';
    }
  });

  // Handle form submission.
  var form = document.getElementById('stripe-form');
  form.addEventListener('submit', function(event) {
    event.preventDefault();

    stripe.createToken(card).then(function(result) {
      if (result.error) {
        // Inform the user if there was an error.
        var errorElement = document.getElementById('card-errors');
        errorElement.textContent = result.error.message;
      } else {
        // Send the token to your server.
        stripeTokenHandler(result.token);
      }
    });
  });

  // Submit the form with the token ID.
  function stripeTokenHandler(token) {
    // Insert the token ID into the form so it gets submitted to the server
    var form = document.getElementById('stripe-form');
    var hiddenInput = document.createElement('input');
    hiddenInput.setAttribute('type', 'hidden');
    hiddenInput.setAttribute('name', 'stripeToken');
    hiddenInput.setAttribute('value', token.id);
    form.appendChild(hiddenInput);

    // Submit the form
    form.submit();
  }

  var currentCardForm = $('.current-card-form');
  var newCardForm = $('.new-card-form');
  var use_default_card = document.querySelector("input[name=use_default_card]");
  use_default_card.addEventListener('change', function() {
    if (this.checked) {
      newCardForm.hide();
      currentCardForm.show()
    } else {
      newCardForm.show();
      currentCardForm.hide()
    }
  })
  ```
  
  the css file:
  
  ```css
  <style>

#stripeBtnLabel {
  font-family: "Helvetica Neue", Helvetica, sans-serif;
  font-size: 16px;
  font-variant: normal;
  padding: 0;
  margin: 0;
  -webkit-font-smoothing: antialiased;
  font-weight: 500;
  display: block;
}

#stripeBtn {
  border: none;
  border-radius: 4px;
  outline: none;
  text-decoration: none;
  color: #fff;
  background: #32325d;
  white-space: nowrap;
  display: inline-block;
  height: 40px;
  line-height: 40px;
  box-shadow: 0 4px 6px rgba(50, 50, 93, .11), 0 1px 3px rgba(0, 0, 0, .08);
  border-radius: 4px;
  font-size: 15px;
  font-weight: 600;
  letter-spacing: 0.025em;
  text-decoration: none;
  -webkit-transition: all 150ms ease;
  transition: all 150ms ease;
  float: left;
  width: 100%
}

button:hover {
  transform: translateY(-1px);
  box-shadow: 0 7px 14px rgba(50, 50, 93, .10), 0 3px 6px rgba(0, 0, 0, .08);
  background-color: #43458b;
}

.stripe-form {
  padding: 5px 30px;
}

#card-errors {
  height: 20px;
  padding: 4px 0;
  color: #fa755a;
}

.stripe-form-row {
  width: 100%;
  float: left;
  margin-top: 5px;
  margin-bottom: 5px;
}

/**
 * The CSS shown here will not be introduced in the Quickstart guide, but shows
 * how you can use CSS to style your Element's container.
 */
.StripeElement {
  box-sizing: border-box;

  height: 40px;

  padding: 10px 12px;

  border: 1px solid transparent;
  border-radius: 4px;
  background-color: white;

  box-shadow: 0 1px 3px 0 #e6ebf1;
  -webkit-transition: box-shadow 150ms ease;
  transition: box-shadow 150ms ease;
}

.StripeElement--focus {
  box-shadow: 0 1px 3px 0 #cfd7df;
}

.StripeElement--invalid {
  border-color: #fa755a;
}

.StripeElement--webkit-autofill {
  background-color: #fefde5 !important;
}

.current-card-form {
  display: none;
}

</style>
  ```
