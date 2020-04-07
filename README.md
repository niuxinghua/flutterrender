此篇討論Flutter's Rendering Pipeline

Flutter architecture overview
engine

exposes a very low level API

very smart about text

knows how to take vector graphics and draw them onto screen

framework (rendering, widgets, material)

is itself composed of several layers
Full pipeline
full-pipeline

流程

用戶輸入驅動視圖更新訊號 (User Input)
動畫進度更新 (Animation)
框架需要開始視圖數據的build (Build widget)
視圖重新佈局，計算個部份的大小，位置 (Layout)
然後進行paint，生成每個視圖的視覺數據 (Paint, 生成Layer tree)
生成的視覺數據不能直接用，因為往往視圖層級非常多，這些數據直接向GPU傳遞很低效，需要進行視圖合成，將多個視圖數據合成為一個
合成的視圖數據其實還是一份向量描述數據 (Composite, in engine)
光柵化幫助把這份數據真正地生成一個一個的像素填充數據 (Rasterize)
將像素填充數據提交給GPU
full-pipeline2

Layout

positioning and sizing elements

Paint

figuring out what those elements actually look like

Composite (in engine)

stacks the together in draw order

Rasterize (in engine)

draw to actual physical pixels

此篇針對Layout, Paint, Composite

Thesis: simple is fast
One-pass, linear-time layout and painting

Both layout and painting use a one-pass linear timed algorithm. So we walk the tree from top to bottom, and then we return up the recursive structure.

In contrast to other systems, for example, where they'll do muti-pass layout, where they'll go to a point in the tree, they walk down to gather some information, and then walk down again to adjust the sizes of things. And if you imagine nesting that, you can very quickly how that becomes N squared work, because you keep recursively walking down the tree and back up.

And so in this system, we want to have a one-pass layout that just walk the tree once, touch every node once on the way down, once on the way up, and then could figure out how big and where everything should be with that.

Simple box constraints can generate expressive layout

We use a very simple constrain model to do layout. So for example when you use a complicated linear constraint model and a whole general purpose constraint solve to figure out where to position things. And that's -- that has some benefits, but we thought what if we could do something much simpler. So our constraint model is basically a box with a min width and min height, and a max width and a max height. And that constraint domain is very easy to solve. If I gave you two of those constraints and ask you to unify them, pretty obvious how to unify them. And so, you can write a very simple constraint solver. And the question is -- our thesis is that's enough to generate very expressive layouts.

Structural repainting using compositing

We do all of our repainting structurally. So instead of tracking which rectangles on the screen are invalid and need to be repainted, we do it structurally in the same structure that the overall tree has. So we can say this sub-tree, it needs to repainted, as opposed to keeping track of which rectangles on the screen. And that turns out to be a very big performance win. They're very good at compositing things.

Rendering
Flutter的Widget tree在實際顯示時會轉換為對應的RenderObject tree實現Layout, Paint。

Rendering 主要提供的功能有

abstract class RendererBinding extends BindingBase with ServicesBinding, SchedulerBinding, HitTestable { ... }
abstract class RenderObject extends AbstractNode with DiagnosticableTreeMixin implements HitTestTarget {
abstract class RenderBox extends RenderObject { ... }
class RenderParagraph extends RenderBox { ... }
class RenderImage extends RenderBox { ... }
class RenderFlex extends RenderBox with ContainerRenderObjectMixin<RenderBox, FlexParentData>,
                                        RenderBoxContainerDefaultsMixin<RenderBox, FlexParentData>,
                                        DebugOverflowIndicatorMixin { ... }
RenderBinding

渲染樹和Flutter engine的膠水層，負責管理幀重繪、窗口尺寸和渲染相關參數變化的監聽

RenderObject

渲染樹中所有節點的基類，定義佈局、繪製和合成相關接口

RenderBox, RenderParagraph, RenderImage, RenderFlex

具體佈局和繪製邏輯的實現類

Flutter介面渲染分為: Layout, Paint, Composite，Layout, Paint在Flutter框架完成，Composite則交由引擎負責。

Layout
Determine size and position

RenderObject
owner
parent
layout()
paint()
parentData
visitChildren
Render object is a pretty abstract concept.

Basically, it has an owner, which is the object that's going to drive the pipeline. And each render object knows what its parent is. But in general, a render object doesn't know anything about its children. All it knows how to is visit its children, which means different render object are free to have different child models. So for example, you can have a render object that has a single unique child, or a render object that has a list of children, or a render object that has several named children. And from the perspective of all the rest of the algorithms, we don't care what the child model is. It's totally up to the render object.

And importantly, there's this concept called parent data, which is a slot on a render object that the parent render object can store data. So if you're familiar with other systems like the web, you aren't allowed to put an inline inside of a block, for example, because block needs to store information on its children that inline doesn't have slots to store. And so, you get these anonymous render objects in your render tree on the web to basically convert the data structures. And so, we avoid that just by having this parent data slot that's managed by the parent instead of by the child. And that'll become important when we talk about positioning.

So importantly about render objects, there's no concept of any coordinate system or anything like that. It just knows that it exists in a tree and has parent.

Layout data flow
layout-data-flow

This is the one-pass data flow. So we walk the tree in a depth first traversal. And we pass down the recursive walk constraints. So from RenderObject's point of view, there are arbitrary constraints. In practice, most people use a thing called a render box. So those boxes will be this box constraints that I talked about. And then up from the bottom of the tree comes the size. So I say here are the constraints on how big you are supposed to be. And you say thank you for those constraints. I'm going to go talk to my children for a little while. And when I'm done, I'm going to go respond and say oh, I figured out I want it to be exactly this size.

RenderBox
size
getMinIntrinsicWidth
getMaxIntrinsicWidth
getMinIntrinsicHeight
getMaxIntrinsicHeight
getDistanceToBaseine
hitTest
So that's sort of abstract, but more concretely it turns out that a very useful coordinate system to work in is Cartesian coordinates, so x and y, and width and height. So there's a specialization of a RenderObject called RenderBox that is much more opinionated about how things are sized and positioned. In particular, it has a size which is a width and height, as opposed to a RenderObject can be an arbitrary thing like a sector on a circle or something. It also adds some intrinsic sizing information, which is it comes up in some esoteric cases.

BoxConstraints
box-constraints

So a box has the idea that we're going to use a particular kind of constraints, these box constraints that I've mentioned. So a box constraint is basically what's depicted on this slide. So there's -- in the width dimension there's a min and max , and in the height dimension there's a min and and a max. And the rule is, if the parent gives you these constraints, you have to be somewhere in this light gray region. You aren't allowed to be too small and you aren't allowed to be too big.

Parent determines size
parent-determines-size

You set the min and the max to the same value, and the min and the max height to the same value. So the child is basically dictated, you have to be exactly this big, because that's the only value that satisfies the constraints.

Width-In, Height-Out
width-in-height-out

You just set the width constraints to be tight and you set the height constraints to be loose. And then the parent is essentially specifying the width and the child gets to report the height that he wants to be.

Height-In, Width-Out
height-in width-out

What's interesting is actually because this model treats width and height, and basically x and y symmetrically, you get the opposite. You get Height-In, Width-Out also arises naturally.

BoxParentData
offset

So here RenderBox knows your own size, but you don't know your position. Your position is here controlled by your parent, in your -- this opaque parent data field that you hold. So what that means is when the parent gets the sizes for all those children, he is then free to reposition them without talking to them again. So without touching them he can move them around, and that turns out to be quite powerful for things like scrolling where you want to scroll a widget and move it around without touching it. You just want to do the minimal amount of work to translate things around.

Flex layout
An example of how layout works.

https://youtu.be/UUfXWzp0-DU?t=707

Relayout boundary
Relayout boundary是在做性能優化

You notice at some point I might give a child a tight constraint, which means the child has to be exactly a certain size. But since his size has to be exactly the one that match the constraint, there's no choice for his size. So whatever crazy layout thing is happing there, that information can't propagate to the rest of the tree. This creates what we call a re-layout boundary. And we compute these implicitly just from watching that algorithm, that constraints over execute. And so it basically says, if somebody in this sub-tree wants to change his size or position, that change is contained in the sub-tree. So when we produce that next frame, we only need to consult this sub-tree. We don't even need to touch the rest of the entire tree, and so that makes things much, much more efficient.

And actually there are different cases.

Constraints.isTight

parentUsesSize == false

When a child -- when parent asks a child for the layout, he supplies a flag that says whether he's going to use the child's size in the rest of this computation. And if he says no, that also create a relayout boundary, because then if the child changes size it doesn't affect anything else, because the parent didn't listen to the size. It was irrelevant from the parent's point of view.

sizedByParent == true

A child can report that his size depends only on his incoming constraints. So it's not -- says, for example, a child that always expands to fill his constraints, he's sized by his parent. Whatever his parents told him his constraints, he immediately knows what his size is. What his children do don't matter, and that also creates a relayout boundary.

Layout order & Paint order
https://youtu.be/UUfXWzp0-DU?t=1401

Paint
Determine visual appearance

Painting phase, so we figured out where everything is and how big it is, but we haven't figured out what it looks like.

流程
將RenderObject轉成Layer
一個RenderObject不一定只對應一個Layer，一個Layer也不一定只對應一個RenderObject
生成Layer後，再透過dart:ui的SceneBuilder掛載到Layer tree
生成Layer tree交給engine
Layers
一個Layer是一份向量數據，掛在到LayerTree後，還會經過Engine的Composite和Rasterize才能交給GPU。

layers

So the complication with painting is that we have to deal with layers. So if you were painting everything to one buffer, then that would be the end of the story. But it turns out that painting things to one buffer is very constraining. So for example, suppose you had a -- suppose this yellow thing in here was a video. So it's something that's going to be drawn by some other part of the system that you don't interact. There's some hardware video code that's just going to write video textures and then you're going to draw them. And you want to draw somethings behind the video and somethings on top of the video. It means you have to divide up your drawing into two different pieces, the part that's below the video and the part that's above, so later, when you're compositing the video, then everything looks correct. So for example, you could draw a play button on the top of the video.

So the tricky thing in painting is basically figuring out in which layer the painting command should go. So conceptually, you can think of these layers as buffers of pixels. We don't actually make pixels out of them. We just keep them as vectors, but you don't have to worry about that too much.

Paint into layers
paint-into-layers

So during the paint phase we go walk the tree in depth layer, and then we paint into these layers. So here the green bubbles are painting into the green layer. Number four here, he's a video or like a child view, or something that needs to be composited in order to paint correctly. So he gets painted into his own layer. And then everything that comes after number four in paint order gets drawn into the red layer.

So the interesting thing to observe is that on the second row on the left, he's got some green and some red aspects to him. So what that meant is, you should imagine that I painted one. I painted the background for two, and then two, painted three and four. And then on the way up he decided he wanted to paint some more things, and then the fifth painting happened. So this -- when he paints after his children, his painting command go into different place, a different layer than when he painted before his children. So they end up in the red layer. And then when we go up to the top, the top has no more painting to do. He goes down to his child, and his child also ends up after the yellow in paint order.

So there's this funny thing where a given render object isn't allocated to unique layer. His painting can actually be split across multiple layers. So this is in contrast to basically -- I'm not aware of any other system that does this. So for example, in Cocoa there's a one-to-one correspondence between UI views and CA layers. You can't split a UI view into multiple CA layers. Similarly on the web, you can't split a single render object into multiple layers. Its painting doesn't work, and there's plenty of bugs because of that.

Paint data flow
RenderObject並不是一一對應的，所以需要paint過程將RenderObject轉成Layer。

paint-data-flow

But in this system we basically do that(render object isn't allocated to unique layer). So the way we do that it is we -- it's not just offsets that we're sending down the tree. There's actually stuff that comes back up in our one-pass walk. In particular, the target layer, so which layer you ought to draw into is something your children tell you as part of painting. So you tell your child go paint yourself over here. And he tells you, hey, you should continue painting in this other layer. So if you were a foreign language person you could think of this as like continuation passing. So he passes the continuation of where you should continue painting. And in that way, the computation of the compositing strategy, so which things are painted into which layer, and the actual recording of the painting commands is unified into one walk that's done in this simple.

So that's nice, but now you see there are all these funny, non-local effects. The fact that this yellow guy had to be composited had an impact on this red guy on some totally other part of the tree. So that would make painting very complicated, because any effect in one part of the tree could have ab effect on some other radically different part of the tree. And so, if this guy said, oh, I want to change my painting, in principle you'd have to repaint everything in the whole universe to make that change happen. And so, while we have this clever idea from the layout that we should introduce these relayout boundaries.

掛載LayerTree
Layer的抽象概念在Flutter是真實存在的，RenderObject在經過paint生成Layer，會經過dart:ui的SceneBuilder掛載到LayerTree。

/// Override this method to upload this layer to the engine. 
/// 
/// The `layerOffset` is the accumulated offset of this layer's parent from the 
/// origin of the builder's coordinate system. 
void addToScene(ui.SceneBuilder builder, Offset layerOffset);
這是一個抽象接口，不同的Layer (ex: ComatinerLayer, TextureLayer) 在繼承抽象類後字型處理邏輯，但無外乎最終都會調用builder .addxxx 這類方法，把傳遞的Layer真正地掛載上去。

Repaint boundary
Repaint boundary是在做性能優化

isRepaintBoundary
alwaysNeedsCompositing
repaint-boundary

So what a repaint boundary does is basically say I'm going to artificially pretend that this child needs its own composited layer. And what that means is it produces -- that the effects in that sub-tree are then contained. They don't affect other parts of the tree. So now this blue guy has to be painted into the blue layer, regardless of whether this yellow guy exists or needs his own layer, or anything crazy like that.

Layer trees
layer-trees

So this diagram is intended to show how the repaint boundary changes the structure of the compositing layer tree. So on the left was our original layer tree. We had the green layer, yellow layer and the red layer. So they're painting here in a pre-ordered traversal.

And on the right, we have what the tree looks like after you've introduced the repaint boundary. So you get this dark blue or black layer that is an artificial layer that we introduced to contain the effects. And now we have this extra blue layer to paint after the black layer.

Composite(in engine)
Update visual appearance, fast

So one benefit you have for breaking your scene up into these composited layers is you can update you visual appearance very fast. So if all you're doing is moving around these layers or changing their offsets or transforms, then you don't have to do any of the rest of the work that we've talked to up to this point. Because you have everything split apart into pieces, you just need to draw those big pieces again. So if you want to move the yellow layer to the right, you don't have to touch anything else. You just move to the right and then re-composite your layers.

Scrolling
scrolling

So a good motivating example for why you want to do this is scrolling. So here, imagine that you have a list that's going to scroll. So the gray things are the different items on the list and the dark gray boundary is the viewpoint, so that's the part of the list that we can see. So as we scroll up here, if you didn't do anything clever, you would have to at least repaint the entire viewpoint every frame of the scroll. Because this pixel change from white to gray, and so that pixel has to repaint. So we go explore the tree until we find a repaint boundary, and then we repaint that whole thing.

Well, that turns out to be less efficient that it could be. And scrolling is a very taxing operation on the system. You want to basically have scrolling be as efficient as possible.

Composited scrolling
解釋composited layers帶來的好處

composited-scrolling

So what you do is you use a separate layer for each of the items in the scrollable list. So here when I move from the first part of the scroll to the second part of the scroll, all I did was shift those boxes up. I didn't have to repaint them. I didn't have to do anything. I just took their -- either their already recorded drawn commands, or if they've been turned into pixels just their pixels and spew them back onto the screen. And as I scroll up I reveal this new item. So the only amount of painting I have to do is when I reveal a new item I have to go create a layer for him, paint them. But now I have him, and as I scroll I don't have to do anymore work. I just have to slide him around. And when this green guy slides at the top, I can reclaim him. And that way I get this nice recycling list view almost for free out of the whole system. So as these buffers or layers become available on the top, they can appear on the bottom. And you only ever have a finite number of them as you scroll through this system. This also connects up to earlier where we talked about where each of these items in the list don't know what their offset is. So their behavior or appearance can't possibly depend on their offset, because they don't know what their offset is. So then I know that I can just move them without talking to them. And so that means I don't have to do -- amount of work. I have to do a composited scroll is essentially very little. And so on like three-year-old devices we can do composited scroll in about one millisecond. That's pretty fast.

Compositing
layer & texture

layer (represented as a vector)
layer經過texturize變成texture (record pixels)
texture是一種增加效率的作法
一旦有了texture，就能夠將pixels直接顯示於screen
如果將layer轉為texture，則會需要GPU memory
In Flutter

draw a layer three times as a vector, then make a texture
as long as you keep drawing it we'll just draw it directly from the texture
Traditionally compositing means I had pixels recorded in a texture. And then I'm going to do is I'm going to blit that texture onto the screen in order. And so we actually do that sometimes, but we don't always do that. So each of these layers can either be represented as a vector, so like a display list, so a list of drawing commands to execute. Or we can bake that list -- that display list into a texture. And then once we have the texture we can blit the pixels directly to the screen.

when do we decide to texturize these layers?

And so we have -- so in other systems they make very strong commitments about this. So Cocoa says every C layer, it's a texture. We're going to have lots of GPU memory. It's going to be OK. Android framework says the opposite. It says I never ever want to make textures. I don't have a lot of memory. I'm going to redraw my display lists from scratch every frame and I'm going to make that really efficient. So this system actually takes a middle approach to these things. So what happens, if you draw the texture three times -- when we draw a layer three times as a vector, we'd be like, we keep drawing this same layer. I bet it's worth making a texture out of it. And the third time we'll first draw it to a texture and then blit it from the texture, and from then on as long as you keep drawing it we'll just draw it directly from the texture. And it turns out that three is kind if a magic number. So if you picked one, then that means you would always draw indirectly through textures. And that would be not efficient in some cases, like imagine a circular progress indicator in material design. So it's arc that keeps changing size and keeps rotating around. It never draws the same frame twice, like ever. So there's no point in drawing it indirectly through a texture. You might as well just draw it directly from its command. But imagine like a drawer that's sliding out. That thing is identical. All that's changing is its offset. So if you capture the drawer into -- in its own repaint boundary and you can translate it in the compositor, then after you've done this a couple times, you're like, hey, I bet this is going to stay like this. And so you can actually just texturize a door as a whole thing and then move it out.
