import XCTest
@testable import TuNombreDeApp // Importa tu módulo principal

// --- Mock/Stub Objects ---

// 1. Un DataSource/Delegate simulado para proporcionar datos y tamaños
class MockLayoutDataSource: NSObject, UICollectionViewDataSource, UICollectionViewDelegateFlowLayout {
    var numberOfSections: Int = 0
    var itemsPerSection: [Int] = []
    var itemSizes: [[CGSize]] = [] // Tamaños por indexPath
    
    var headerSizes: [CGSize] = [] // Tamaños por sección

    // MARK: DataSource
    func numberOfSections(in collectionView: UICollectionView) -> Int {
        return numberOfSections
    }

    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        guard section < itemsPerSection.count else { return 0 }
        return itemsPerSection[section]
    }

    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        // No necesitamos una celda real, solo cumplir el protocolo
        return UICollectionViewCell()
    }

    func collectionView(_ collectionView: UICollectionView, viewForSupplementaryElementOfKind kind: String, at indexPath: IndexPath) -> UICollectionReusableView {
         // No necesitamos una vista real
         return UICollectionReusableView()
    }

    // MARK: DelegateFlowLayout
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize {
        guard indexPath.section < itemSizes.count,
              indexPath.item < itemSizes[indexPath.section].count else {
            return CGSize(width: 50, height: 50) // Tamaño por defecto
        }
        return itemSizes[indexPath.section][indexPath.item]
    }

    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, referenceSizeForHeaderInSection section: Int) -> CGSize {
         guard section < headerSizes.count else {
            return .zero // Sin header por defecto
         }
         return headerSizes[section]
    }

    // Añade implementaciones para otros métodos de FlowLayout si tu layout los necesita
    // (ej. minimumInteritemSpacingForSectionAt, etc.) o confía en los del layout.
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, minimumLineSpacingForSectionAt section: Int) -> CGFloat {
        return (collectionViewLayout as? UICollectionViewFlowLayout)?.minimumLineSpacing ?? 10
    }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, minimumInteritemSpacingForSectionAt section: Int) -> CGFloat {
        return (collectionViewLayout as? UICollectionViewFlowLayout)?.minimumInteritemSpacing ?? 10
    }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, insetForSectionAt section: Int) -> UIEdgeInsets {
        return (collectionViewLayout as? UICollectionViewFlowLayout)?.sectionInset ?? .zero
    }

}

// 2. Un CollectionView simulado muy básico
// Nota: Esto es muy simplificado. Frameworks de Mocking harían esto más robusto.
class MockCollectionView: UICollectionView {
    var mockDataSource: UICollectionViewDataSource?
    var mockDelegate: UICollectionViewDelegateFlowLayout? // Usamos FlowLayout aquí
    var mockContentOffset: CGPoint = .zero
    var mockAdjustedContentInset: UIEdgeInsets = .zero
    var mockBounds: CGRect = .zero

    // Sobrescribir propiedades clave para devolver valores controlados
    override var dataSource: UICollectionViewDataSource? {
        get { return mockDataSource }
        set { mockDataSource = newValue}
    }
    override var delegate: UICollectionViewDelegate? { // El delegate general
        get { return mockDelegate }
        set { mockDelegate = newValue as? UICollectionViewDelegateFlowLayout }
    }
    override var contentOffset: CGPoint {
        get { return mockContentOffset }
        set { mockContentOffset = newValue }
    }
     override var adjustedContentInset: UIEdgeInsets {
        get { return mockAdjustedContentInset }
        set { mockAdjustedContentInset = newValue }
    }
    override var bounds: CGRect {
        get { return mockBounds }
        set { mockBounds = newValue }
    }

    // Métodos que el layout podría necesitar
    override var numberOfSections: Int {
        return mockDataSource?.numberOfSections?(in: self) ?? 0
    }
    override func numberOfItems(inSection section: Int) -> Int {
        return mockDataSource?.collectionView(self, numberOfItemsInSection: section) ?? 0
    }

    // Inicializador requerido
    init() {
        // Necesitamos un layout dummy inicial, lo reemplazaremos
        super.init(frame: .zero, collectionViewLayout: UICollectionViewFlowLayout())
    }
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}

// --- Test Case ---

class StickyHeaderFlowLayoutTests: XCTestCase {

    var layout: StickyHeaderFlowLayout!
    var mockCollectionView: MockCollectionView!
    var mockDataSource: MockLayoutDataSource!

    // Tamaños y configuración comunes para las pruebas
    let defaultHeaderHeight: CGFloat = 50
    let defaultCellHeight: CGFloat = 80
    let defaultCellWidth: CGFloat = 100
    let defaultSpacing: CGFloat = 10
    let collectionViewWidth: CGFloat = 320 // Ancho típico para pruebas
    let stickySection = 1 // La sección que debe ser pegajosa

    override func setUpWithError() throws {
        try super.setUpWithError()

        layout = StickyHeaderFlowLayout()
        layout.stickyHeaderSection = stickySection // Configurar sección pegajosa
        layout.minimumLineSpacing = defaultSpacing
        layout.minimumInteritemSpacing = defaultSpacing
        layout.sectionInset = UIEdgeInsets(top: defaultSpacing, left: defaultSpacing, bottom: defaultSpacing, right: defaultSpacing)


        mockCollectionView = MockCollectionView()
        mockCollectionView.mockBounds = CGRect(x: 0, y: 0, width: collectionViewWidth, height: 500) // Tamaño visible
        mockCollectionView.collectionViewLayout = layout // Asignar layout al mock CV
        mockCollectionView.mockAdjustedContentInset = .zero // Asumir sin insets para simplificar cálculos

        mockDataSource = MockLayoutDataSource()
        mockCollectionView.dataSource = mockDataSource
        mockCollectionView.delegate = mockDataSource // El mismo objeto maneja ambos

        // Asignar el mock CV de vuelta al layout (¡MUY IMPORTANTE!)
        layout.collectionView = mockCollectionView
    }

    override func tearDownWithError() throws {
        layout = nil
        mockCollectionView = nil
        mockDataSource = nil
        try super.tearDownWithError()
    }

    // --- Helper para configurar datos del DataSource ---
    func setupMockData(sections: Int, itemsPerSec: Int, itemWidth: CGFloat, itemHeight: CGFloat, headerHeight: CGFloat) {
        mockDataSource.numberOfSections = sections
        mockDataSource.itemsPerSection = Array(repeating: itemsPerSec, count: sections)
        mockDataSource.itemSizes = Array(repeating: Array(repeating: CGSize(width: itemWidth, height: itemHeight), count: itemsPerSec), count: sections)
        mockDataSource.headerSizes = Array(repeating: CGSize(width: collectionViewWidth, height: headerHeight), count: sections)
    }

    // --- Helper para obtener atributos de un tipo/indexPath específico ---
    func findAttributes(in attributes: [UICollectionViewLayoutAttributes]?, kind: String?, indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        return attributes?.first { $0.indexPath == indexPath && ($0.representedElementKind == kind || (kind == nil && $0.representedElementKind == nil)) }
    }

    // --- Tests ---

    func testInitialLayout_HeaderPositionsAndZIndex() {
        // Arrange
        setupMockData(sections: 3, itemsPerSec: 2, itemWidth: defaultCellWidth, itemHeight: defaultCellHeight, headerHeight: defaultHeaderHeight)
        mockCollectionView.contentOffset = .zero // Sin scroll inicial

        // Act
        let attributes = layout.layoutAttributesForElements(in: mockCollectionView.bounds)

        // Assert
        // Header 0 (no pegajoso)
        let header0Attrs = findAttributes(in: attributes, kind: UICollectionView.elementKindSectionHeader, indexPath: IndexPath(item: 0, section: 0))
        XCTAssertNotNil(header0Attrs, "Header 0 debería existir")
        XCTAssertEqual(header0Attrs?.frame.origin.y ?? -1, layout.sectionInset.top, accuracy: 0.1, "Header 0 Y inicial incorrecto")
        XCTAssertEqual(header0Attrs?.zIndex ?? -1, layout.standardHeaderZIndex, "Header 0 zIndex inicial incorrecto")

        // Header 1 (potencialmente pegajoso)
        let header1Attrs = findAttributes(in: attributes, kind: UICollectionView.elementKindSectionHeader, indexPath: IndexPath(item: 0, section: stickySection))
        XCTAssertNotNil(header1Attrs, "Header 1 debería existir")
        // Calcular Y esperado: Y de header 0 + altura header 0 + altura celdas sección 0 + espaciados
        let section0ContentHeight = defaultHeaderHeight + ceil(CGFloat(2) / floor((collectionViewWidth - layout.sectionInset.left - layout.sectionInset.right + defaultSpacing) / (defaultCellWidth + defaultSpacing))) * (defaultCellHeight + defaultSpacing) - defaultSpacing // Cálculo aproximado altura contenido sección 0
        let expectedHeader1Y = layout.sectionInset.top + section0ContentHeight + layout.sectionInset.bottom + layout.sectionInset.top // Sumar insets entre secciones
        // Nota: El cálculo exacto del FlowLayout es complejo, esta es una aproximación. Es mejor obtenerlo del propio layout si es posible.
        // Opcionalmente, obtener el frame directo de super:
        let originalHeader1Attrs = layout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: IndexPath(item: 0, section: stickySection))
        XCTAssertEqual(header1Attrs?.frame.origin.y ?? -1, originalHeader1Attrs?.frame.origin.y ?? -2, accuracy: 0.1, "Header 1 Y inicial debería ser el original")
        // zIndex debería ser el pegajoso porque el layout lo asigna incluso si no está pegado aún
        XCTAssertEqual(header1Attrs?.zIndex ?? -1, layout.stickyHeaderZIndex, "Header 1 zIndex inicial incorrecto")

        // Celda 0,0
        let cell00Attrs = findAttributes(in: attributes, kind: nil, indexPath: IndexPath(item: 0, section: 0))
        XCTAssertNotNil(cell00Attrs, "Celda 0,0 debería existir")
        XCTAssertEqual(cell00Attrs?.frame.origin.y ?? -1, layout.sectionInset.top + defaultHeaderHeight + defaultSpacing, accuracy: 0.1, "Celda 0,0 Y inicial incorrecta")
        XCTAssertEqual(cell00Attrs?.zIndex ?? -1, 0, "Celda 0,0 zIndex inicial incorrecto")
    }

    func testScrolling_WhenStickyHeaderBecomesSticky() {
        // Arrange
        setupMockData(sections: 3, itemsPerSec: 5, itemWidth: defaultCellWidth, itemHeight: defaultCellHeight, headerHeight: defaultHeaderHeight)
        // Necesitamos la Y original del header pegajoso
        guard let originalHeader1Attrs = layout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: IndexPath(item: 0, section: stickySection)) else {
            XCTFail("No se pudieron obtener los atributos originales del Header 1")
            return
        }
        let stickyTriggerY = originalHeader1Attrs.frame.origin.y - mockCollectionView.adjustedContentInset.top // El offset que hace que se pegue

        mockCollectionView.contentOffset = CGPoint(x: 0, y: stickyTriggerY + 10) // Scroll justo después de que se pegue

        // Act
        let attributes = layout.layoutAttributesForElements(in: mockCollectionView.bounds) // El rect visible ahora empieza en stickyTriggerY + 10

        // Assert
        let header1Attrs = findAttributes(in: attributes, kind: UICollectionView.elementKindSectionHeader, indexPath: IndexPath(item: 0, section: stickySection))
        XCTAssertNotNil(header1Attrs, "Header 1 debería estar visible cuando está pegajoso")

        let expectedStickyY = mockCollectionView.contentOffset.y + mockCollectionView.adjustedContentInset.top
        XCTAssertEqual(header1Attrs?.frame.origin.y ?? -1, expectedStickyY, accuracy: 0.1, "Header 1 Y debería estar pegado al tope (offset + inset)")
        XCTAssertEqual(header1Attrs?.zIndex ?? -1, layout.stickyHeaderZIndex, "Header 1 zIndex debería ser el pegajoso")

         // Verificar que una celda de la sección 1 está *debajo* del header pegajoso
         let cell10Attrs = findAttributes(in: attributes, kind: nil, indexPath: IndexPath(item: 0, section: stickySection))
         XCTAssertNotNil(cell10Attrs, "Celda 1,0 debería estar visible")
         XCTAssertGreaterThan(cell10Attrs?.frame.origin.y ?? -1, header1Attrs?.frame.maxY ?? 0, "Celda 1,0 debería estar debajo del Header 1 pegajoso")
         XCTAssertEqual(cell10Attrs?.zIndex ?? -1, 0, "Celda 1,0 zIndex incorrecto")

        // Verificar que Header 0 (si aún está en el rect teórico) tiene zIndex normal
        // El rect de prueba quizá no incluya Header 0 aquí. Podríamos probar pidiendo el atributo directamente.
         let directHeader0Attrs = layout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: IndexPath(item: 0, section: 0))
          XCTAssertEqual(directHeader0Attrs?.zIndex ?? -1, layout.standardHeaderZIndex, "Header 0 zIndex directo incorrecto")


    }

     func testScrolling_FarPastStickyHeader() {
        // Arrange
        setupMockData(sections: 3, itemsPerSec: 10, itemWidth: defaultCellWidth, itemHeight: defaultCellHeight, headerHeight: defaultHeaderHeight)
        guard let originalHeader1Attrs = layout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: IndexPath(item: 0, section: stickySection)) else {
            XCTFail("No se pudieron obtener los atributos originales del Header 1")
            return
        }
        let farScrollY = originalHeader1Attrs.frame.origin.y + 500 // Scroll muy abajo

        mockCollectionView.contentOffset = CGPoint(x: 0, y: farScrollY)
         // Ajustar el rect visible para la prueba
        var visibleRect = mockCollectionView.bounds
        visibleRect.origin.y = farScrollY
        mockCollectionView.bounds = visibleRect // Actualizar bounds para que layoutAttributesForElements use el rect correcto

        // Act
        let attributes = layout.layoutAttributesForElements(in: visibleRect)

        // Assert
        // Header 1 (pegajoso) debería seguir en la parte superior del rect visible
        let header1Attrs = findAttributes(in: attributes, kind: UICollectionView.elementKindSectionHeader, indexPath: IndexPath(item: 0, section: stickySection))
        XCTAssertNotNil(header1Attrs, "Header 1 debería estar presente muy abajo en el scroll")
        let expectedStickyY = farScrollY + mockCollectionView.adjustedContentInset.top
        XCTAssertEqual(header1Attrs?.frame.origin.y ?? -1, expectedStickyY, accuracy: 0.1, "Header 1 Y debería seguir pegado al tope")
        XCTAssertEqual(header1Attrs?.zIndex ?? -1, layout.stickyHeaderZIndex, "Header 1 zIndex debería ser el pegajoso")

        // Header 2 (si es visible) debería estar debajo y con zIndex normal
        let header2Attrs = findAttributes(in: attributes, kind: UICollectionView.elementKindSectionHeader, indexPath: IndexPath(item: 0, section: 2))
         if header2Attrs != nil { // Puede que no esté en el rect visible aún
             XCTAssertGreaterThan(header2Attrs!.frame.origin.y, header1Attrs!.frame.maxY, "Header 2 debería estar debajo del Header 1 pegajoso")
             XCTAssertEqual(header2Attrs!.zIndex, layout.standardHeaderZIndex, "Header 2 zIndex incorrecto")
         }
     }

    func testDirectAttributeRequest_StickyHeaderWhenSticky() {
         // Arrange
        setupMockData(sections: 3, itemsPerSec: 5, itemWidth: defaultCellWidth, itemHeight: defaultCellHeight, headerHeight: defaultHeaderHeight)
        guard let originalHeader1Attrs = layout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: IndexPath(item: 0, section: stickySection)) else {
            XCTFail("No se pudieron obtener los atributos originales del Header 1")
            return
        }
        let stickyTriggerY = originalHeader1Attrs.frame.origin.y - mockCollectionView.adjustedContentInset.top
        mockCollectionView.contentOffset = CGPoint(x: 0, y: stickyTriggerY + 50) // Hacerlo pegajoso

        // Act
        let directAttrs = layout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: IndexPath(item: 0, section: stickySection))

        // Assert
        XCTAssertNotNil(directAttrs, "Debería poder obtener atributos directos para header pegajoso")
        let expectedStickyY = mockCollectionView.contentOffset.y + mockCollectionView.adjustedContentInset.top
        XCTAssertEqual(directAttrs?.frame.origin.y ?? -1, expectedStickyY, accuracy: 0.1, "Y directo incorrecto para header pegajoso")
        XCTAssertEqual(directAttrs?.zIndex ?? -1, layout.stickyHeaderZIndex, "ZIndex directo incorrecto para header pegajoso")

    }

}







# NWSTokenView

[![CI Status](http://img.shields.io/travis/Appmazo/NWSTokenView.svg?style=flat)](https://travis-ci.org/Appmazo/NWSTokenView)
[![Version](https://img.shields.io/cocoapods/v/NWSTokenView.svg?style=flat)](http://cocoapods.org/pods/NWSTokenView)
[![License](https://img.shields.io/cocoapods/l/NWSTokenView.svg?style=flat)](http://cocoapods.org/pods/NWSTokenView)
[![Platform](https://img.shields.io/cocoapods/p/NWSTokenView.svg?style=flat)](http://cocoapods.org/pods/NWSTokenView)
[![Beerpay](https://beerpay.io/Appmazo/NWSTokenView/badge.svg?style=beer-square)](https://beerpay.io/Appmazo/NWSTokenView)
[![Beerpay](https://beerpay.io/Appmazo/NWSTokenView/make-wish.svg?style=flat-square)](https://beerpay.io/Appmazo/NWSTokenView?focus=wish)

![NWSTokenView Demo](/Screenshots/NWSTokenViewExample.gif)

# Introduction
NWSTokenView is a flexible UIView subclass that shows a collection of objects in a similar manner to the Messages app. 

## Why is it different from others?
NWSTokenView’s main difference when compared to other similar libraries is the fact that it allows you to easily create your own style tokens via XIB files or programmatically without all the headaches. NWSTokenView does come with a default token style you can use.

## Installation

NWSTokenView is available through [CocoaPods](http://cocoapods.org). To install
it, simply add the following line to your Podfile:

```ruby
pod "NWSTokenView"
```

## Usage

To run the example project, clone the repo, and run `pod install` from the Example directory first.

# How to use

## Import

```swift
import NWSTokenView
```

## Subclass NWSToken

You can create your own customized tokens by subclassing the NWSToken class. In the example, you can see how this done in the NWSImageToken class:

```swift
public class NWSImageToken: NWSToken
{
    @IBOutlet weak var imageView: UIImageView!
    @IBOutlet weak var titleLabel: UILabel!
    
    public class func initWithTitle(title: String, image: UIImage? = nil) -> NWSImageToken?
    {
        …set UI here…
    }
}
```

## Protocol Conformance

```swift
class ViewController: UIViewController, NWSTokenViewDataSource, NWSTokenViewDelegate
{
     override func viewDidLoad()
     {
          super.viewDidLoad()
          tokenView.dataSource = self
          tokenView.delegate = self
     }
}
```

## Data Source Implementation

Return the token data to display in the tokenView.

The number of tokens to display:

```swift
func numberOfTokensForTokenView(tokenView: NWSTokenView) -> Int
{
    return tokens.count
}
```

The insets for the tokenView:

```swift
func insetsForTokenView(tokenView: NWSTokenView) -> UIEdgeInsets?
{
    return UIEdgeInsetsMake(5, 5, 5, 5)
}
```

The title for the tokenView:

```swift
func titleForTokenViewLabel(tokenView: NWSTokenView) -> String?
{
    return "To:"
}
```

The font for the tokenView label:

```swift
func fontForTokenViewLabel(tokenView: NWSTokenView) -> UIFont?
{
    return UIFont.systemFont(ofSize: 14.0)
}
```

The text color for the tokenView label:

```swift
func textColorForTokenViewLabel(tokenView: NWSTokenView) -> UIColor?
{
    return UIColor.black
}
```

The placeholder text for the tokenView when there are no tokens:

```swift
func titleForTokenViewPlaceholder(tokenView: NWSTokenView) -> String?
{
    return "Search contacts..."
}
```

The font for the tokenView textView:

```swift
func fontForTokenViewTextView(tokenView: NWSTokenView) -> UIFont?
{
    return UIFont.systemFont(ofSize: 14.0)
}
```

The text color for the tokenView textView when there are no tokens:

```swift
func textColorForTokenViewPlaceholder(tokenView: NWSTokenView) -> UIColor?
{
    return UIColor.lightGray
}
```

The text color for the tokenView textView:

```swift
func textColorForTokenViewTextView(tokenView: NWSTokenView) -> UIColor?
{
    return UIColor.black
}
```

The custom view for the tokens:

```swift
func tokenView(tokenView: NWSTokenView, viewForTokenAtIndex index: Int) -> UIView?
{
    let contact = contacts[Int(index)]
    if let token = NWSToken.initWithTitle(contact.name, image: contact.image)
    {
        return token
    }
    return nil
}
```

## Delegate Implementation

Return the behaviors for the token view.

Notifies you when a token was selected:

```swift
func tokenView(tokenView: NWSTokenView, didSelectTokenAtIndex index: Int)
{
    // NOTE - If getting the token itself using ‘tokenForIndex()’, be sure to convert the token to your own subclass.
    // Example:
    // var token = tokenView.tokenForIndex(index) as! NWSImageToken
}
```
   
Notifies you when a token was deselected: 

```swift
func tokenView(tokenView: NWSTokenView, didDeselectTokenAtIndex index: Int)
{
    // NOTE - If getting the token itself using ‘tokenForIndex()’, be sure to convert the token to your own subclass.
    // Example:
    // var token = tokenView.tokenForIndex(index) as! NWSImageToken
}
```
    
Notifies you when a token was deleted (i.e. selected then backspaced/overwritten/etc.):

```swift
func tokenView(tokenView: NWSTokenView, didDeleteTokenAtIndex index: Int)
{
    // Do something
}
```

Notifies you when the token view’s textField becomes the first responder:

```swift
func tokenView(tokenViewDidBeginEditing: NWSTokenView)
{
    // Do something
}
```

Notifies you when the token view’s textField resigns the first responder: 

```swift
func tokenViewDidEndEditing(tokenView: NWSTokenView)
{
    // Do something
}
```

Notifies you when the token view’s textField’s text is changed:  

```swift  
func tokenView(tokenView: NWSTokenView, didChangeText text: String)
{
    // Do something
}
```

Notifies you when the token view’s textField’s text is returned:  

```swift    
func tokenView(tokenView: NWSTokenView, didEnterText text: String)
{
    // Do something    
}
```

Notifies you when the token view’s content size has changed (i.e. new line added): 

```swift     
func tokenView(tokenView: NWSTokenView, contentSizeChanged size: CGSize)
{
    // Do something
}
```

Notifies you when the token view finished loading all tokens:  

```swift        
func tokenView(tokenView: NWSTokenView, didFinishLoadingTokens tokenCount: Int)
{
    // Do something
}
```

## Author

James Hickman, jhickman@appmazo.com

## License

The MIT License (MIT)

Copyright (c) 2015 Appmazo, LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
