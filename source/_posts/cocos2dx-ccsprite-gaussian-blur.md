---
title: 【cocos2d-x】对CCSprite进行高斯模糊
date: 2014-01-12 00:00:00
tags:
  - tech
  - cpp
  - cocos2dx
---

你可以从下面的目录找到示例的源代码：

> cocos2d-x-2.2.1\samples\Cpp\TestCpp\Classes\ShaderTest

SpriteBlur类用于实现高斯模糊，但并没有定义在ShaderTest.h中，打开ShaderTest.cpp，大概在488行有这个类的定义和实现：

```cpp
// ShaderBlur

class SpriteBlur : public CCSprite
{
public:
    ~SpriteBlur();
    void setBlurSize(float f);
    bool initWithTexture(CCTexture2D* texture, const CCRect&  rect);
    void draw();
    void initProgram();
    void listenBackToForeground(CCObject *obj);

    static SpriteBlur* create(const char *pszFileName);

    CCPoint blur_;
    GLfloat    sub_[4];

    GLuint    blurLocation;
    GLuint    subLocation;
};
```

实现：

```cpp
SpriteBlur::~SpriteBlur()
{
    CCNotificationCenter::sharedNotificationCenter()->removeObserver(this, EVENT_COME_TO_FOREGROUND);
}

SpriteBlur* SpriteBlur::create(const char *pszFileName)
{
    SpriteBlur* pRet = new SpriteBlur();
    if (pRet && pRet->initWithFile(pszFileName))
    {
        pRet->autorelease();
    }
    else
    {
        CC_SAFE_DELETE(pRet);
    }
    
    return pRet;
}

void SpriteBlur::listenBackToForeground(CCObject *obj)
{
    setShaderProgram(NULL);
    initProgram();
}

bool SpriteBlur::initWithTexture(CCTexture2D* texture, const CCRect& rect)
{
    if( CCSprite::initWithTexture(texture, rect) ) 
    {
        CCNotificationCenter::sharedNotificationCenter()->addObserver(this,
                                                                      callfuncO_selector(SpriteBlur::listenBackToForeground),
                                                                      EVENT_COME_TO_FOREGROUND,
                                                                      NULL);
        
        CCSize s = getTexture()->getContentSizeInPixels();

        blur_ = ccp(1/s.width, 1/s.height);
        sub_[0] = sub_[1] = sub_[2] = sub_[3] = 0;

        this->initProgram();
        
        return true;
    }

    return false;
}

void SpriteBlur::initProgram()
{
    GLchar * fragSource = (GLchar*) CCString::createWithContentsOfFile(
                                CCFileUtils::sharedFileUtils()->fullPathForFilename("Shaders/example_Blur.fsh").c_str())->getCString();
    CCGLProgram* pProgram = new CCGLProgram();
    pProgram->initWithVertexShaderByteArray(ccPositionTextureColor_vert, fragSource);
    setShaderProgram(pProgram);
    pProgram->release();
    
    CHECK_GL_ERROR_DEBUG();
    
    getShaderProgram()->addAttribute(kCCAttributeNamePosition, kCCVertexAttrib_Position);
    getShaderProgram()->addAttribute(kCCAttributeNameColor, kCCVertexAttrib_Color);
    getShaderProgram()->addAttribute(kCCAttributeNameTexCoord, kCCVertexAttrib_TexCoords);
    
    CHECK_GL_ERROR_DEBUG();
    
    getShaderProgram()->link();
    
    CHECK_GL_ERROR_DEBUG();
    
    getShaderProgram()->updateUniforms();
    
    CHECK_GL_ERROR_DEBUG();
    
    subLocation = glGetUniformLocation( getShaderProgram()->getProgram(), "substract");
    blurLocation = glGetUniformLocation( getShaderProgram()->getProgram(), "blurSize");
    
    CHECK_GL_ERROR_DEBUG();
}

void SpriteBlur::draw()
{
    ccGLEnableVertexAttribs(kCCVertexAttribFlag_PosColorTex );
    ccBlendFunc blend = getBlendFunc();
    ccGLBlendFunc(blend.src, blend.dst);

    getShaderProgram()->use();
    getShaderProgram()->setUniformsForBuiltins();
    getShaderProgram()->setUniformLocationWith2f(blurLocation, blur_.x, blur_.y);
    getShaderProgram()->setUniformLocationWith4fv(subLocation, sub_, 1);

    ccGLBindTexture2D( getTexture()->getName());

    //
    // Attributes
    //
#define kQuadSize sizeof(m_sQuad.bl)
    long offset = (long)&m_sQuad;

    // vertex
    int diff = offsetof( ccV3F_C4B_T2F, vertices);
    glVertexAttribPointer(kCCVertexAttrib_Position, 3, GL_FLOAT, GL_FALSE, kQuadSize, (void*) (offset + diff));

    // texCoods
    diff = offsetof( ccV3F_C4B_T2F, texCoords);
    glVertexAttribPointer(kCCVertexAttrib_TexCoords, 2, GL_FLOAT, GL_FALSE, kQuadSize, (void*)(offset + diff));

    // color
    diff = offsetof( ccV3F_C4B_T2F, colors);
    glVertexAttribPointer(kCCVertexAttrib_Color, 4, GL_UNSIGNED_BYTE, GL_TRUE, kQuadSize, (void*)(offset + diff));


    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

    CC_INCREMENT_GL_DRAWS(1);
}

void SpriteBlur::setBlurSize(float f)
{
    CCSize s = getTexture()->getContentSizeInPixels();

    blur_ = ccp(1/s.width, 1/s.height);
    blur_ = ccpMult(blur_,f);
}
```

好了，直接copy到你的program里面，不过有一点需要注意，就是他这个只能用一个文件(图片)create，如果需要用一个Texture初始化(因为有时候需要模糊即时的sprite)，可以稍微改装一下，加一个函数：

```cpp
static SpriteBlur* createWithTexture(CCTexture2D *pTexture);
```

实现：

```cpp
SpriteBlur* SpriteBlur::createWithTexture(CCTexture2D *pTexture)
{
    CCAssert(pTexture != NULL, "Invalid texture for sprite");

    CCRect rect = CCRectZero;
    rect.size = pTexture->getContentSize();

	SpriteBlur* pRet = new SpriteBlur();
	if (pRet && pRet->initWithTexture(pTexture,rect))
    {
        pRet->autorelease();
    }
    else
    {
        CC_SAFE_DELETE(pRet);
    }
    
    return pRet;
}
```

用法：

```cpp
SpriteBlur *bluredSpr = SpriteBlur::createWithTexture(tex);
bluredSpr->setPosition(ccp(sz.width/2,sz.height/2));
bluredSpr->setBlurSize(0.9f); // 这里稍微设小一点
addChild(bluredSpr);
```

效果：

![](224243_MkOW_580940.png)

注意：

他需要一个fsh文件（具体看它的实现），似乎是叠texture用的，找到example_Blur.fsh放到你的Resources\Shaders目录下
