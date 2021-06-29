# <center> Back-end </div>

###### <hr/>

* ## Laravel
    * case 1) This part of <b>ProductController.php</b> from `imeds.ir`, product has many `sku` (subProduct)
      , `variants` method dispatched this product and the subProduct's (good).
      `productVariantChecker()` check `sku` requested is belongTo product, Finally `variants` method hashed productId
      with
      `kmp()` helper function and go ahead for show into the `product.show`.
      > <small>for more review: https://www.imeds.ir/product/imp-1357G324/%D8%A8%D8%A7%D9%86%D8%AF%D8%A7%DA%98-%D8%AA%D8%B1%D9%82%D9%88%D9%87 </small>
      ```php
      class ProductController extends Controller
      {
          public function variants(VariantRequest $request, $product, $category): RedirectResponse
          {
              $product = Product::findOrFail(idGenerator($product, 'down'));
              $category = Category::findOrFail(idGenerator($category, 'down'));
      
              if (productVariantChecker($request, $product)) {
                  $good = productVariantChecker($request, $product);
                  return redirect()->route('products.show', [
                      KMP($good, $category, 'variants'),
                      KMPT($product)
                  ]);
              }
              throw new NotFoundHttpException();
          }
      }
      ```
      ```php
      function productVariantChecker($request, $product)
      {
          foreach ($product->goods as $good) {
            $data = [];
            foreach ($good->values as $value)
                $data[] = $value->id;
            $data = array_diff($data, $request->values);
            if (empty($data))
                return $good;
        }
         return false;
      }
      ```
       ```php
      function KMP($product, $category, $option = null)
      {
            if (empty($option)) {
                if (isProductAvailable($product))
                    $good = _default('good', $product);
                else
                    $good = $product->good;
            } else
                $good = $product;
      
            $newCategoryId = idGenerator($category->id, 'up');
            $newGoodId = idGenerator($good->id, 'up');
      
            return _KMP . $newGoodId . _delimiter . $newCategoryId;
      }
      ```

    * case 2) This part of <b>CartController.php</b> from `imeds.ir`, store card into the session using `Redis` driver,
      store cart into the database using `Mysql` driver.
      > <small>for more review: https://www.imeds.ir/carts </small>
        ```php
            public function storeToSession($request, $good, $option = null)
            {
                $carts = Session::get('cart.items');
        
                if ($carts) {
                    if (array_key_exists($good->id, $carts)) {
                        if ($option)
                            $carts[$good->id]->count = $request->count;
                        else
                            $carts[$good->id]->count += $request->count;
                    } else {
                        $carts = Arr::add($carts, $good->id, $good);
                        $carts[$good->id]->count = $request->count;
                    }
                } else {
                    $carts = Arr::add($carts, $good->id, $good);
                    $carts[$good->id]->count = $request->count;
                }
        
                if ($this->countChecker($carts[$good->id]->count, $good))
                    $carts[$good->id]->count = maxQuantity($good);
        
                Session::put('cart.items', $carts);
        
                return $carts;
            }
        ```
        ```php
            public function storeToDatabase($request, $good, $option = null)
            {
                $user = auth()->user();
                $targetCart = $user->carts($good);
        
                if ($targetCart) {
                    if ($option) {
                        $this->authorize('canUpdateCart', $targetCart);
                        $targetCart->count = $request->count;
                    } else
                        $targetCart->count += $request->count;
                    if ($this->countChecker($targetCart->count, $good))
                        $targetCart->count = maxQuantity($good);
                    $targetCart->update(['count' => $targetCart->count]);
                } else {
                    if ($this->countChecker($request->count, $good))
                        $request->count = maxQuantity($good);
                    $user->carts()->create(['good_id' => $good->id, 'count' => $request->count]);
                }
                return $user->carts;
            }
        ```

    * case 3) This part of <b>PaymentController.php</b> from `imeds.ir`, SoapClient on `WSDL (behPardakh Melat)` web
      services, dispatch information to go on `Melat Payment gateway` page.
      > <small>for more review: https://bpm.shaparak.ir/pgwchannel/services/pgw?wsdl </small>
      ```php
      public function __construct()
      {
        // try to connect to web service
        try
            $this->soapClient = new SoapClient(config('app.gateway_wsdl'));
        catch (\SoapFault $e){
            dd($e->getMessage());
        }
        
        $this->gateway_terminal_id = config('app.gateway_terminal_id');
        $this->gateway_username = config('app.gateway_username');
        $this->gateway_user_password = config('app.gateway_user_password');
        $this->localDate = date("Ymd");
        $this->localTime = date("His");
        $this->additionalData = ' ';
        $this->payerId = 0;
      }
      
       public function _payRequest($order)
      {
        try {
            $result = $this->soapClient->bpPayRequest($this->parameters('payRequest', $order));
            $result = explode(',', $result->return);
            if ($result[0] == 0) {
                auth()->user()->transactions()->create([
                    'order_id' => $order->id,
                    'result_return' => $result[0],
                    'status_id' => config('app.orderStatusId.placeOrderAndGoToPaymentGatewayId'),
                    'ref_id' => $result[1],
                ]);
                $order->update([
                    'status_id' => config('app.orderStatusId.placeOrderAndGoToPaymentGatewayId'),
                ]);
                return view('gateway.index', ['refId' => $result[1]]);
            } else {
                auth()->user()->transactions()->create([
                    'order_id' => $order->id,
                    'result_return' => $result[0],
                    'status_id' => config('app.orderStatusId.gatewayUnavailableId'),
                ]);
                $order->update([
                    'status_id' => config('app.orderStatusId.gatewayUnavailableId'),
                ]);
                return 'درحال حاضر اتصال به درگاه پرداخت امکان پذیر نیست مجدد اقدام کنید';
            }
            } catch (\Throwable $e) {
                return $e->getMessage();
          }
      }
      ```

    * case 4) Work With <b>240 Million</b> records from `Iranian Air Force (NDA)` with `mysql partition`
      & `laravel lazyCollection`.
      > <small>for more about how create partition in mysql using laravel: https://github.com/lucabecchetti/laravel-mysql-partition </small>
      <b>UserController.php</b> `laravel lazyCollection`
      ```php
        public function jobs(User $user): LazyCollection
        {
            $jobs = DB::transaction(function () use ($user) {
                $query = $user->jobs()->where('is_completed', true);
                if ($user->id == 2)
                    return $query->where('status', 1)->where('main', false);
                elseif ($user->id == 3)
                    return $query->where('status', 2)->where('main', true);
                $query = $query->where('status', 0);
                return $query->get();
            });
        
        
            /* Lazy Collection and allows you to use the existing collection methods.
             * In this case, no query is being generated!.
             * The make() method will help to chain the entire section with other helper collections
             * we take $records with chunk($records) method. 1/240 million
             */
            $jobs = LazyCollection::make(function () use ($jobs) {
                yield $jobs;
            })->chunk(1000000)->each(function ($job) use ($user) {
                Mail::to($user->email)->send(jobFechted::getInstance('Fetched job id: ' . $job->id));
            });
    
            return $jobs;
        }            
      ```
      <b>2020_11_5_062029_create_jobs_table.php</b> `mysql partition`
      ```php
        public function up()
        {
          Schema::create('jobs', function (Blueprint $table) {
                $table->id();
                $table->unsignedBigInteger('user_id')->index();
                $table->unsignedBigInteger('status_id')->index();
                $table->string('title');
                $table->boolean('main')->index();
                $table->timestamps();
        
                $table->foreign('user_id')
                    ->references('id')
                    ->on('users')
                    ->onDelete('NO ACTION');
                
                $table->foreign('status_id')
                    ->references('id')
                    ->on('status')
                    ->onDelete('NO ACTION');
                });
      
                // Force autoincrement of one field in composite primary key
                Schema::forceAutoIncrement('jobs', 'id');
        
                // Create partition by range for jobs status
                Schema::partitionByRange('jobs', 'status', [
                    new Partition('status0', Partition::RANGE_TYPE, 0),
                    new Partition('status1', Partition::RANGE_TYPE, 1),
                    new Partition('status2', Partition::RANGE_TYPE, 2)
                ]);
          } 
      ```

    * case 4) Laravel <b>LiveWire</b>.
        ```php
        class ProductOffer extends Component
        {

           public $product;
           public $currentOffer;
        
            public function mount()
            {
                $this->currentOffer = $this->product->offers->first();
            }
        
            public function bids(offer $offer)
            {
                $this->currentOffer = $offer;
            }
        
            public function render()
            {
                $offers = $this->product->offers->sortByDesc(function ($offer) {
                        return $offer->bids->where('product_id', $this->product->id)->count();
                     });
                return view('livewire.product-offer', [
                    'offers' => $offers,
                    'currentOffer' => $this->currentOffer,
                ]);
            }
        }
        ```
      <b>Blade Template</b>
        ```xhtml
            <a wire:click="bids({{$offer->id}})" class="link">
                {!! $offer->title !!}
            </a>
        ```


* ## php ~ ajax
    * > Please look my repository in github: https://github.com/arefSajjadi/simplePhpChat `:)`




      
# <center> Front-end </div>

###### <hr/>

* ## javascript (native)
    * case 1) Review of code on `mobile menu` manager & `ajax search` dispatcher.
      > <small>mobile menu, source: https://imeds.ir </small>
        ```javascript
            // mobilemenu
            $(function () {
                const body = $('body');
                const mobilemenu = $('.mobilemenu');
        
                if (mobilemenu.length) {
                    const open = function () {
                        const bodyWidth = body.width();
                        body.css('overflow', 'hidden');
                        body.css('paddingRight', (body.width() - bodyWidth) + 'px');
        
                        mobilemenu.addClass('mobilemenu--open');
                    };
                    const close = function () {
                        body.css('overflow', '');
                        body.css('paddingRight', '');
        
                        mobilemenu.removeClass('mobilemenu--open');
                    };
        
        
                    $('.mobile-header__menu-button').on('click', function () {
                        open();
                    });
                    $('.mobilemenu__backdrop, .mobilemenu__close').on('click', function () {
                        close();
                    });
                }
            });
        
            // tooltips
            $(function () {
                $('[data-toggle="tooltip"]').tooltip({trigger: 'hover'});
            });
            
            // layout switcher
            $(function () {
                $('.layout-switcher__button').on('click', function () {
                    const layoutSwitcher = $(this).closest('.layout-switcher');
                    const productsView = $(this).closest('.products-view');
                    const productsList = productsView.find('.products-list');
        
                    layoutSwitcher.find('.layout-switcher__button').removeClass('layout-switcher__button--active');
                    $(this).addClass('layout-switcher__button--active');
        
                    productsList.attr('data-layout', $(this).attr('data-layout'));
                    productsList.attr('data-with-features', $(this).attr('data-with-features'));
                });
            });
        ```
      > <small>search with ajax for more action and about this use `searchBar` on top this page: https://imeds.ir </small>
      ```javascript
        // search with ajax
        $('.search').each(function (index, element) {
            let xhr;
            const search = $(element);
            const categories = search.find('.search__categories');
            const input = search.find('.search__input');
            const suggestions = search.find('.search__suggestions');
            const outsideClick = function (event) {
                // If the inner element still has focus, ignore the click.
                if ($(document.activeElement).closest('.search').is(search)) {
                    return;
                }
        
                search
                    .not($(event.target).closest('.search'))
                    .removeClass('search--suggestions-open');
            };
            const setSuggestion = function (html) {
                if (html) {
                    suggestions.html(html);
                }
                search.toggleClass('search--has-suggestions', !!html);
            };
        
            search.on('focusout', function () {
                setTimeout(function () {
                    if (document.activeElement === document.body) {
                        return;
                    }
        
                    // Close suggestions if the focus received an external element.
                    search
                        .not($(document.activeElement).closest('.search'))
                        .removeClass('search--suggestions-open');
                }, 10);
            });
            input.on('input', function () {
                if (xhr) {
                    // Abort previous AJAX request.
                    xhr.abort();
                }
        
                if (input.val()) {
                    // AJAX REQUEST HERE.
                    xhr = $.ajax({
                        url: '/search',
                        data: {
                            value: $(this).val()
                        },
                        beforeSend: function () {
                            $("#searchLoading*").show();
                        },
                        complete: function () {
                            $("#searchLoading*").hide();
                        },
                        success: function (data) {
                            xhr = null;
                            setSuggestion(data);
                        }
                    });
                } else {
                    // Remove suggestions.
                    setSuggestion('');
                }
            });
            input.on('focus', function () {
                search.addClass('search--suggestions-open');
            });
            categories.on('focus', function () {
                search.removeClass('search--suggestions-open');
            });
        
            document.addEventListener('click', outsideClick, true);
            touchClick(document, outsideClick);
        
            if (input.is(document.activeElement)) {
                input.trigger('focus').trigger('input');
            }
        });
      ```

* ## vue.js (axios)
    * case 1) create `follow|unfollow` toggle button for use on same social network applications.
        ```vue
        <template>
        <div>
    
            <button class="btn btn-primary ml-4" @click="FollowUser" v-text="buttonText"></button>
    
        </div>
        </template>
      
        <script>
            export default {
                props: ['user_id' , 'follows'],
        
                mounted() {
                    console.log('Component mounted.')
                },
        
                data: function(){
                    return{
                        status: this.follows,
                    }
                },
        
                methods: {
                    FollowUser() {
                        axios.post('/follow/' + this.user_id)
                        .then(response => {
                            this.status = ! this.status;
                            console.log(response.data);
                        });
                    }
                },
        
                computed:{
                    buttonText(){
                        return (this.status) ? 'Unfollow' : 'Follow'
                    }
                }
        
            }
        </script>
        ```







# <center> Final </center>

###### <hr/>

>>> <center>Boldly, that I have more expertise and ability in the field of Back-end. However, i didn't work with front-end framework. </center>






`Title: part-of-code-experience`
`Author: Aref sajjadi`
`created_at: 2021/jun/28~29`
`updated_at: 2021/jun/29`














