# Testing unitario en Angular

Una prueba unitaria es una forma de comprobar el correcto funcionamiento de una unidad de código, para asegurar que cada unidad funcione correctamente y eficientemente por separado, además de verificar que el código haga lo que tiene que hacer.

## Componentes

Un componente combina una plantilla HTML y una clase de TypeScript. De modo que es necesario comprobar que tanto la clase como la plantilla funcionen correctamente.

Dichas pruebas requieren crear el elemento host del componente en el DOM del navegador, como lo hace Angular, e investigar la interacción de la clase del componente con el DOM como lo describe su plantilla.

Angular TestBedfacilita este tipo de pruebas. Pero en muchos casos, probar solo la clase del componente , sin la participación del DOM, puede validar gran parte del comportamiento del componente de una manera sencilla y más obvia.

Las pruebas de clase de componentes deben mantenerse muy limpias y simples. Debes probar una sola unidad. A primera vista, deberías poder entender lo que está probando test.

Vamos a poner un ejemplo: componente LightswitchComponentque enciende y apaga una luz (representada por un mensaje en pantalla) cuando el usuario hace clic en el botón.

```ts
@Component({
  selector: "lightswitch-comp",
  template: ` <button class="custom-button" (click)="clicked()">
      Click me!
    </button>
    <span>{{ message }}</span>`,
})
export class LightswitchComponent {
  isOn = false;
  clicked() {
    this.isOn = !this.isOn;
  }
  get message() {
    return `The light is ${this.isOn ? "On" : "Off"}`;
  }
}
```

Podemos ver que este sencillo componente tiene en la clase unas variables y métodos públicos, que es lo que normalmente siempre vamos a testear. En este caso, las variables públicas tienen un valor por defecto el cual cambia al llamar a la función clicked, así que solo hay que testear ese funcionamiento:

```ts
describe("LightswitchComp", () => {
  it("#clicked() should toggle #isOn", () => {
    const comp = new LightswitchComponent();
    expect(comp.isOn).toBe(false);
    comp.clicked();
    expect(comp.isOn).toBe(true);
  });

  it('#clicked() should set #message to "is on"', () => {
    const comp = new LightswitchComponent();
    expect(comp.message).toMatch(/is off/i);
    comp.clicked();
    expect(comp.message).toMatch(/is on/i);
  });
});
```

Dicho esto, las pruebas de solo clase pueden informarte sobre el comportamiento de la clase, pero no pueden decirte si el componente se procesará correctamente, responderá a la entrada y los gestos del usuario, o si se integrará con sus componentes padre e hijo. A nuesto test anterior deberiamos añadir siLightswitch.clicked() está vinculado a algo tal que el usuario pueda invocarlo, y si el Lightswitch.message se muestra. Vamos a probarlo con la configuración del TestBed en el beforeEach:

```ts
describe('LightswitchComp', () => {
  let component: LightswitchComp;
  let fixture: ComponentFixture<LightswitchComp>;

  beforeEach(() => {
    TestBed.configureTestingModule({declarations: [LightswitchComp]});
    fixture = TestBed.createComponent(LightswitchComp);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeDefined();
  });
  it('#clicked() should toggle #isOn', () => {
    expect(component.isOn).toBe(false);
    component.clicked();
    expect(component.isOn).toBe(true');
  });

  it('#clicked() should set #message to "is on"', () => {
    expect(component.message).toMatch(/is off/i);
    component.clicked();
    expect(component.message).toMatch(/is on/i);
  });
  it('should contain "The light is Off" if "isOn" is false', () => {
   const element: HTMLElement = fixture.nativeElement;
    const span = element.querySelector('span')!;
   expect(span.textContent).toEqual('The light is Off');
  });
  it('should contain "The light is On" if "isOn" is true', () => {
    component.clicked();
   const element: HTMLElement = fixture.nativeElement;
    const span = element.querySelector('span')!;
   expect(span.textContent).toEqual('The light is On');
  });
  it('should call "clicked" method if user implement a click event in the button', () => {
    const button: HTMLButtonElement =
          fixture.debugElement.query(By.css('.custom-button')).nativeElement;
    button.click();
   expect(component.isOn).toBe(true);
  });
});
```

Pongamos el caso de que el componente tenga un método asíncrono para mostrar una alerta del mensaje:

```ts
@Component({
  selector: "lightswitch-comp",
  template: ` <button class="custom-button" (click)="clicked()">
      Click me!
    </button>
    <button class="custom-button-2" (click)="showAlert()">Show alert!</button>
    <span>{{ message }}</span>`,
})
export class LightswitchComponent {
  isOn = false;
  clicked() {
    this.isOn = !this.isOn;
  }
  get message() {
    return `The light is ${this.isOn ? "On" : "Off"}`;
  }
  showAlert() {
    new Promise((resolve) => {
      resolve(`Alert: ${this.message}`);
    }).then((value: string) => {
      alert(value);
    });
  }
}
```

El test que usaríamos para probar esta secuencia asíncrona necesitaría el siguiente it:

```ts
it(
  "should show an alert if user clicks on button to show the alert",
  waitForAsync(() => {
    spyOn(window, "alert");
    const button: HTMLButtonElement = fixture.debugElement.query(
      By.css(".custom-button-2")
    ).nativeElement;
    button.click();
    fixture.whenStable().then(() => {
      fixture.detectChanges();
      expect(window.alert).toHaveBeenCalledWith("The light is Off");
    });
  })
);
```

En este caso hemos usado un waitForAsync, unfixture.whenStable() y un fixture.detectChanges, junto a un espía para hacer el expect del alert. De esta forma estamos probando que el flujo asíncrono que tiene este método se ejecute. Pero el problema con este flujo es que estamos introduciendo una espera real en el test, y esto puede hacerlo lento en determinados casos. Así que vamos a usar fakeAsync.

fakeAsync es una zona especial que nos permite probar código asíncrono de forma síncrona. A diferencia de la zona original que realiza algún trabajo y delega la tarea al navegador o a Node.js, fakeAsync almacena en búfer cada tarea internamente y expone una API que nos permite decidir cuándo se debe ejecutar la tarea.

```ts
it("should show an alert if user clicks on button to show the alert", fakeAsync(() => {
  spyOn(window, "alert");
  const button: HTMLButtonElement = fixture.debugElement.query(
    By.css(".custom-button-2")
  ).nativeElement;
  button.click();
  tick();
  fixture.detectChanges();
  expect(window.alert).toHaveBeenCalledWith("The light is Off");
}));
```

## Servicios

Los servicios son unidades de código a testear más sencillas que los componentes, ya que solo hay que probar la clase.

Vamos a realizar algunas pruebas unitarias sincrónicas y asincrónicas de un ValueService:

```ts
// Straight Jasmine testing without Angular's testing support
describe("ValueService", () => {
  let service: ValueService;
  beforeEach(() => {
    service = new ValueService();
  });

  it("#getValue should return real value", () => {
    expect(service.getValue()).toBe("real value");
  });

  it("#getObservableValue should return value from observable", (done: DoneFn) => {
    service.getObservableValue().subscribe((value) => {
      expect(value).toBe("observable value");
      done();
    });
  });

  it("#getPromiseValue should return value from a promise", (done: DoneFn) => {
    service.getPromiseValue().then((value) => {
      expect(value).toBe("promise value");
      done();
    });
  });
});
```

La mayoría de servicios se basan en la inyección de dependencia (DI). Cuando un servicio tiene un servicio dependiente, la DI encuentra o crea ese servicio dependiente. Y si ese servicio dependiente tiene sus propias dependencias, DI también las encuentra o las crea. Para testear servicios que tengan dependencias inyectadas, al menos hay que pensar en el primer nivel de dependencias.

Vamos a probar esta clase con el TestBed de Angular:

```ts
describe("ValueService", () => {
  let service: ValueService;
  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [ValueService] });
    service = TestBed.inject(ValueService);
  });

  it("#getValue should return real value", () => {
    expect(service.getValue()).toBe("real value");
  });
});
```

## Peticiones

Los servicios de datos que realizan llamadas HTTP a servidores remotos generalmente inyectan y delegan al HttpClientservicio Angular para llamadas XHR.

Las interacciones extendidas entre un servicio de datos y el HttpClientpueden ser complejas y difíciles de burlar con los espías.

El HttpClientTestingModulepuede hacer que estos escenarios de prueba más manejable, hacen más fácil la gestión de las peticiones mockeadas usando el servicio HttpTestingController.

Pongamos un ejemplo de un servicio:

```ts
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class CoursesService {
  constructor(private http: HttpClient) {}

  addCourse(course: any): Observable<any> {
    return this.http.post<any>(
      `http://localhost:8089/topics/${course.topicId}/courses`,
      course
    );
  }

  getCoursesByTopic(topicId: any): Observable<any> {
    return this.http.get(`http://localhost:8089/topics/${topicId}/courses`);
  }
}
```

El test sería lo siguiente:

```ts
import { TestBed } from "@angular/core/testing";
import { CoursesService } from "./courses.service";
import {
  HttpClientTestingModule,
  HttpTestingController,
} from "@angular/common/http/testing";

describe("CoursesService", () => {
  let httpTestingController: HttpTestingController;
  let service: CoursesService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [CoursesService],
      imports: [HttpClientTestingModule],
    });

    // We inject our service (which imports the HttpClient) and the Test Controller
    httpTestingController = TestBed.get(HttpTestingController);
    service = TestBed.get(CoursesService);
  });

  afterEach(() => {
    httpTestingController.verify();
  });

  // Angular default test added when you generate a service using the CLI
  it("should be created", () => {
    expect(service).toBeTruthy();
  });
  describe("#addCourse()", () => {
    it("returned Observable should match the right data", () => {
      const mockCourse = {
        name: "Chessable",
        description: "Space repetition to learn chess, backed by science",
      };
      service.addCourse({ topicId: 1 }).subscribe((courseData) => {
        expect(courseData.name).toEqual("Chessable");
      });
      const req = httpTestingController.expectOne(
        "http://localhost:8089/topics/1/courses"
      );
      expect(req.request.method).toEqual("POST");
      req.flush(mockCourse);
    });
  });
  describe("#getCoursesByTopic", () => {
    it("returned Observable should match the right data", () => {
      const mockCourses = [
        {
          name: "Chessable",
          description: "Space repetition to learn chess, backed by science",
        },
        {
          name: "ICC",
          description: "Play chess online",
        },
      ];
      service.getCoursesByTopic(1).subscribe((coursesData) => {
        expect(coursesData[0].name).toEqual("Chessable");
        expect(coursesData[0].description).toEqual(
          "Space repetition to learn chess, backed by science"
        );
        expect(coursesData[1].name).toEqual("ICC");
        expect(coursesData[1].description).toEqual("Play chess online");
      });
      const req = httpTestingController.expectOne(
        "http://localhost:8089/topics/1/courses"
      );
      req.flush(mockCourses);
    });
  });
});
```

¿Por qué falseamos datos y hacemos que testeen que esos datos se gestionen en las requests? Porque un test unitario es para un trozo de código aislado que no va a conectarse a un servidor remoto, de modo que hay que mockear esos datos y testear acciones y eventos de las clases de forma como si un servidor remoto ya estuviese conectado. No nos interesa qué dato recibimos del servidor, sino qué es lo que ocurre en nuestra clase con ese datos.

Otra forma es mockear el servicio entero:

```ts
class CoursesMockService {
  addCourse(course: any): Observable<any> {
    return of(course);
  }
  getCoursesByTopic(topicId: any) {
    return of([
      {
        name: "Chessable",
        description: "Space repetition to learn chess, backed by science",
      },
      {
        name: "ICC",
        description: "Play chess online",
      },
    ]);
  }
}

// ...
TestBed.configureTestingModule({
  // ...
  providers: [
    {
      provide: CoursesService,
      useClass: CoursesMockService,
    },
  ],
  // ...
});
//...
```

## Pipes

Debido a que una pipe es una clase que tiene un método, transform (que manipula el valor de entrada en un valor de salida transformado), es más fácil de probar sin ninguna utilidad de prueba de Angular.

A continuación se muestra un ejemplo de cómo debería verse una prueba de pipe:

```ts
describe("TroncaturePipe", () => {
  it("create an instance", () => {
    const pipe = new TroncaturePipe(); // * pipe instantiation
    expect(pipe).toBeTruthy();
  });

  it("truncate a string if its too long (>20)", () => {
    const pipe = new TroncaturePipe();
    const value = pipe.transform("123456789123456789456666123");
    expect(value.length).toBe(20);
  });
});
```

## Directivas

Una directiva de atributo modifica el comportamiento de un elemento. Por lo tanto, se puede testear como una pipe donde solo prueba sus métodos, o con un componente host donde se verifica si cambia correctamente su comportamiento.

Aquí hay un ejemplo de cómo probar una directiva con un componente host:

```ts
// * Host component:
@Component({
  template: `<div [appPadding]="2">Test</div>`,
})
class HostComponent {}
@NgModule({
  declarations: [HostComponent, PaddingDirective],
  exports: [HostComponent],
})
class HostModule {}

// * Test suite:
describe("PaddingDirective", () => {
  let component: HostComponent;
  let element: HTMLElement;
  let fixture: ComponentFixture<HostComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [CommonModule, HostModule], // * we import the host module
    }).compileComponents();

    fixture = TestBed.createComponent(HostComponent);
    component = fixture.componentInstance;
    element = fixture.nativeElement;

    fixture.detectChanges(); // * so the directive gets appilied
  });

  it("should create a host instance", () => {
    expect(component).toBeTruthy();
  });

  it("should add padding", () => {
    // * arrange
    const el = element.querySelector("div");
    // * assert
    expect(el.style.padding).toBe("2rem"); // * we check if the directive worked correctly
  });
});
```

## Espías

Los espías son una manera fácil de verificar si se llamó a una función o de proporcionar un valor de retorno personalizado. Por ejemplo:

```ts
it("should do something", () => {
  // arrange
  const service = TestBed.get(dataService);
  const spyOnMethod = spyOn(service, "saveData").and.callThrough();
  // act
  component.onSave();
  // assert
  expect(spyOnMethod).toHaveBeenCalled();
});
```

Se puede indicar un espía a una propiedad de un objeto con spyOnProperty, y hacer que devuelva un valor al llamar a esa propiedad:

```ts
// ts
export const detectLanguage = () =>
  navigator.language ||
  (Array.isArray(navigator.languages) && navigator.languages[0]);

// spec
describe("detectLanguage", () => {
  it("Should return en-US if language from navigator is en-US", () => {
    spyOnProperty(navigator, "language").and.returnValue("en-US");
    expect(detectLanguage()).toBe("en-US");
  });
});
```

También se puede crear un espía sobre un método:

```ts
// ts
export const randomBoolean = () => Math.random() >= 0.5;

// spec
describe("randomBoolean", () => {
  it("Should return true if random value is 0.7", () => {
    spyOn(Math, "random").and.returnValue(0.7);
    expect(randomBoolean()).toBeTruthy();
  });
});
```
