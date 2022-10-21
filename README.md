# Как передать параметры при редиректе в angular #

Существуют сайты, у которых есть отдельные версии для декстопа и мобилок. При этом для пользователя важна возможность перейти с одной версии на другую так, чтобы при переходе он попал на экран, аналогичный тому, с которого он этот переход совершил. При этом чаще всего берется текущий url (например,"somedomen.com/main/somepathname"), корректируется нужным образом домен ( на "m.somedomen.com/main/somepathname") и совершается переход. В большинстве случаев, если в обоих версиях использовались идентичные роуты для аналогичных друг другу компонентов, все отработает корректно. Но что если роуты отличаются? Работать с самим механизмом перехода не вариант, потому что тогда приложение, из которого пользователь уходит будет знать слишком много о приложении, куда нужно перейти, что некошерно. Поэтому всё должно заключаться в обработке попытки загрузить некорректные пути на "целевом" сайте.

Все примеры, описанные в этот статье можно посмотреть и пощупать в [этой песочнице](https://stackblitz.com/edit/angular-ivy-kuzpjr?file=src%2Fapp%2Fapp.routing.module.ts). Попытки загрузить страницы с некорректными роутами имитированы с помощью кнопок-ссылок на главной страницы.

Начнем c самого простого случая - пути просто не совпадают. Здесь поможет простой редирект:

    // app.routing.module.ts

    const routes: Routes = [
      ...,
      { path: 'non-exist-route', redirectTo: 'from-non-exist-route' },
      ...
    ]

## Передача параметров в paramsMap ##

Перейдем к ситуации поинтересней: допустим на сайте "исхода" компонент имеет свой роут, а его двойник в нашем приложении отображается родителем исходя из параметра в paramsMap в роуте:

    // info-with-params.component.ts

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

    // info-with-params.component.html

    ...
      <app-load-doc
        [hidden]="step !== 'addDoc'"
        (returnToDetails)="returnToDetails()"
      >
      </app-load-doc>
    ...

В этом случае может выручить [urlMatcher](https://angular.io/api/router/UrlMatcher):


    // app.routing.module.ts

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