# Как передать параметры при редиректе в angular #

Всем привет, меня зовут Павел, я работаю в кампании "Takeoff-staff" frontend-разработчиком. В этой статье я хочу немного рассказать о редиректе в angular.

Существуют сайты, у которых есть отдельные версии для декстопа и мобилок. При этом для пользователя важна возможность перейти с одной версии на другую так, чтобы при переходе он попал на экран, аналогичный тому, с которого он этот переход совершил. При этом чаще всего берется текущий url (например,"somedomen.com/main/somepathname"), корректируется нужным образом домен ( на "m.somedomen.com/main/somepathname") и совершается переход. В большинстве случаев, если в обоих версиях использовались идентичные роуты для аналогичных друг другу компонентов, все отработает корректно. Но что если роуты отличаются? Работать с самим механизмом перехода не вариант, потому что тогда приложение, из которого пользователь уходит будет знать слишком много о приложении, куда нужно перейти, что некошерно. Поэтому всё должно заключаться в обработке попытки загрузить некорректные пути на "целевом" сайте.

Все примеры, описанные в этот статье можно посмотреть и пощупать в [этой песочнице](https://stackblitz.com/edit/angular-ivy-kuzpjr?file=src%2Fapp%2Fapp.routing.module.ts). Попытки загрузить страницы с некорректными роутами имитированы с помощью кнопок-ссылок на главной странице.

Начнем c самого простого случая - пути просто не совпадают. Здесь поможет простой редирект:

    // src/app/app.routing.module.ts

    const routes: Routes = [
      ...,
      { path: 'non-exist-route', redirectTo: 'from-non-exist-route' },
      ...
    ]

## Передача параметров в paramsMap ##

Перейдем к ситуации поинтересней: допустим на сайте "исхода" компонент имеет свой роут, а его двойник в нашем приложении отображается родителем исходя из параметра в paramsMap в роуте:

    // src/app/components/info-with-params.component.ts

    export class InfoWithParamsComponent implements OnInit {
      ...
        ngOnInit() {
          this.route.paramMap
            .pipe(
              map((params: ParamMap) => params.get('doc')),
              untilDestroyed(this)
            )
            .subscribe((doc) => {
              if (doc === '1') {
                this.step = 'addDoc';
              } else {
                this.step = 'details';
              }
            });
        }
      ...
    }

    // src/app/components/info-with-params.component.html

    ...
      <app-load-doc
        [hidden]="step !== 'addDoc'"
        (returnToDetails)="returnToDetails()"
      >
      </app-load-doc>
    ...

В этом случае может выручить [urlMatcher](https://angular.io/api/router/UrlMatcher):

    // src/app/app.routing.module.ts

    function matcherForDocs(
      segments: UrlSegment[],
      group: UrlSegmentGroup,
      route: Route
    ): UrlMatchResult | null {
      if (segments[0].path === 'add-documents-with-matcher') {
        const url = new UrlSegment('info-params', { doc: '1' });

        return {
          consumed: [url],
        };
      }
      return null;
    }

    const routes: Routes = [
      ...,
      { matcher: matcherForDocs, component: InfoWithParamsComponent },
      ...
    ]

## Передача query-параметров ##

Но если родитель использует queryParams, то, к сожалению, матчер тут уже не поможет (именно с такой ситуацией я столкнулся на реальном проекте и появилось желание поделиться своим опытом в этой статье). В этом случае можно сделать редирект в самом компоненте:

    // info-with-query-params1.component.ts
    export class InfoWithQueryParamsComponent1 implements OnInit {
      ...

      ngOnInit() {
        if (this.route.snapshot.url[0].path === 'add-documents-query-params-1') {
          this.router.navigate(['/info-query-params-1'], {
            queryParams: {
              doc: 1,
            },
          });

          // тут нужно избежать отработки остальной логики при инициализации компонента, иначе это произойдёт дважды
          return;
        }

        ...
      }

Однако более изящным решением мне кажется в данной ситуации будет использование гарда:

    // src/app/guards/
    @Injectable()
    export class RedirectGuard implements CanActivate {
      constructor(private _router: Router) {}

      public canActivate(route: ActivatedRouteSnapshot): boolean {
        if (route.routeConfig.path === 'add-documents-query-params-2') {
          this._router.navigate(['/info-query-params-2'], {
            queryParams: {
              doc: 1,
            },
          });
          return false;
        }
        return true;
      }
    }
    
    // src/app/app.routing.module.ts

    const routes: Routes = [
      ...,
        {
          path: 'add-documents-query-params-2',
          canActivate: [RedirectGuard],
          component: InfoWithQueryParamsComponent2,
        },
      ...
    ]

Как видно в логах консоли родитель в этом случае инициализируется один раз, как и должен, и отображает нужный нам компонент.

Надеюсь, этот материал был Вам полезен и поможет в будущем, если Вы столкнетесь с подобными кейсами в процессе разработки.

