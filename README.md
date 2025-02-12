# html-smooth-gallery-like-phone-app
ant design html相册。图片相册会自然缩放，由3个每行变成二个每行

```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>手势控制相册</title>
    <script src="https://cdn.jsdelivr.net/npm/react@18.2.0/umd/react.production.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/react-dom@18.2.0/umd/react-dom.production.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/dayjs@1.11.10/dayjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@babel/standalone@7.23.6/babel.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/antd@5.12.2/dist/antd.min.js"></script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/antd@5.12.2/dist/reset.css">
    
    <style>
        .gallery-container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
            min-height: 100vh;
            touch-action: none; /* 防止默认的触摸行为 */
        }

        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 24px;
            flex-wrap: wrap;
            gap: 12px;
        }

        .gallery-grid {
            position: relative;
            width: 100%;
            min-height: 600px;
            touch-action: none;
        }

        .zoom-indicator {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.7);
            color: white;
            padding: 20px;
            border-radius: 12px;
            font-size: 16px;
            pointer-events: none;
            opacity: 0;
            transition: opacity 0.3s;
            z-index: 1000;
        }

        .zoom-indicator.visible {
            opacity: 1;
        }

        .image-card {
            position: absolute;
            background: #fff;
            border-radius: 12px;
            overflow: hidden;
            box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
            transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
        }

        .image-wrapper {
            position: relative;
            padding-top: 75%;
            overflow: hidden;
            background: #f5f5f5;
        }

        .image-info {
            padding: 12px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-top: 1px solid #f0f0f0;
        }

        .resolution {
            font-size: 14px;
            color: #1a1a1a;
            font-weight: 500;
        }

        .size {
            font-size: 13px;
            color: #666;
        }

        @media (max-width: 768px) {
            .gallery-container {
                padding: 12px;
            }
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { Image, Button, message } = antd;

        const App = () => {
            const [images, setImages] = React.useState([]);
            const [columns, setColumns] = React.useState(3);
            const [loading, setLoading] = React.useState(true);
            const containerRef = React.useRef(null);
            const [containerWidth, setContainerWidth] = React.useState(0);
            const [showZoomIndicator, setShowZoomIndicator] = React.useState(false);
            const [indicatorText, setIndicatorText] = React.useState('');
            const touchStartRef = React.useRef({ distance: 0, columns: 3 });

            const formatSize = (bytes) => {
                if (bytes < 1024) return bytes + ' B';
                else if (bytes < 1048576) return (bytes / 1024).toFixed(1) + ' KB';
                else return (bytes / 1048576).toFixed(1) + ' MB';
            };

            // 计算两个触摸点之间的距离
            const getDistance = (touch1, touch2) => {
                return Math.hypot(
                    touch1.clientX - touch2.clientX,
                    touch1.clientY - touch2.clientY
                );
            };

            // 处理触摸开始事件
            const handleTouchStart = (e) => {
                if (e.touches.length === 2) {
                    touchStartRef.current = {
                        distance: getDistance(e.touches[0], e.touches[1]),
                        columns: columns
                    };
                }
            };

            // 处理触摸移动事件
            const handleTouchMove = (e) => {
                if (e.touches.length === 2) {
                    const currentDistance = getDistance(e.touches[0], e.touches[1]);
                    const scale = currentDistance / touchStartRef.current.distance;
                    
                    setShowZoomIndicator(true);

                    if (scale < 0.7 && columns < 4) {
                        setIndicatorText('继续缩小以增加列数');
                        if (scale < 0.6) {
                            setColumns(prev => Math.min(prev + 1, 4));
                            touchStartRef.current.distance = currentDistance;
                            message.success(`已切换为${columns + 1}列布局`);
                            setShowZoomIndicator(false);
                        }
                    } else if (scale > 1.3 && columns > 2) {
                        setIndicatorText('继续放大以减少列数');
                        if (scale > 1.4) {
                            setColumns(prev => Math.max(prev - 1, 2));
                            touchStartRef.current.distance = currentDistance;
                            message.success(`已切换为${columns - 1}列布局`);
                            setShowZoomIndicator(false);
                        }
                    } else {
                        setIndicatorText('✨ 双指缩放以改变布局');
                    }
                }
            };

            // 处理触摸结束事件
            const handleTouchEnd = () => {
                setShowZoomIndicator(false);
            };

            const calculatePositions = () => {
                if (!containerWidth) return [];
                
                const gap = 20;
                const itemWidth = (containerWidth - (gap * (columns - 1))) / columns;
                const itemHeight = itemWidth * 0.75 + 44;

                return images.map((image, index) => {
                    const column = index % columns;
                    const row = Math.floor(index / columns);
                    
                    return {
                        ...image,
                        style: {
                            width: itemWidth,
                            height: itemHeight,
                            transform: `translate(${column * (itemWidth + gap)}px, ${row * (itemHeight + gap)}px)`,
                        }
                    };
                });
            };

            const loadImages = async () => {
                setLoading(true);
                try {
                    const imagePromises = Array(12).fill().map(async () => {
                        const timestamp = Date.now();
                        const random = Math.random();
                        return {
                            url: `https://api.algobook.info/v1/randomimage?category=nature&t=${timestamp}-${random}`,
                            size: Math.floor(Math.random() * 5000000),
                            resolution: '1920 x 1080'
                        };
                    });

                    const loadedImages = await Promise.all(imagePromises);
                    setImages(loadedImages);
                } catch (error) {
                    message.error('加载图片失败，请刷新重试');
                } finally {
                    setLoading(false);
                }
            };

            const handleResize = React.useCallback(() => {
                if (containerRef.current) {
                    setContainerWidth(containerRef.current.offsetWidth);
                }
            }, []);

            React.useEffect(() => {
                loadImages();
                handleResize();
                window.addEventListener('resize', handleResize);
                return () => window.removeEventListener('resize', handleResize);
            }, []);

            const positionedImages = calculatePositions();

            return (
                <div className="gallery-container">
                    <div className="header">
                        <h1>自然风光相册</h1>
                        <div style={{ display: 'flex', gap: '12px' }}>
                            <Button
                                type="primary"
                                onClick={() => {
                                    setColumns(prev => prev === 3 ? 2 : 3);
                                    message.success(`已切换为${columns === 3 ? '两' : '三'}列布局`);
                                }}
                            >
                                切换为{columns === 3 ? '两' : '三'}列布局
                            </Button>
                            <Button onClick={loadImages}>刷新图片</Button>
                        </div>
                    </div>
                    
                    <div
                        className="gallery-grid"
                        ref={containerRef}
                        onTouchStart={handleTouchStart}
                        onTouchMove={handleTouchMove}
                        onTouchEnd={handleTouchEnd}
                    >
                        {positionedImages.map((image, index) => (
                            <div
                                key={index}
                                className="image-card"
                                style={image.style}
                            >
                                <div className="image-wrapper">
                                    <Image
                                        src={image.url}
                                        alt={`自然风光 ${index + 1}`}
                                        style={{
                                            position: 'absolute',
                                            top: 0,
                                            left: 0,
                                            width: '100%',
                                            height: '100%',
                                            objectFit: 'cover'
                                        }}
                                        preview={{
                                            mask: (
                                                <div style={{
                                                    fontSize: '14px',
                                                    color: '#fff',
                                                    background: 'rgba(0,0,0,0.5)',
                                                    padding: '8px',
                                                    borderRadius: '4px'
                                                }}>
                                                    点击预览
                                                </div>
                                            )
                                        }}
                                    />
                                </div>
                                <div className="image-info">
                                    <span className="resolution">{image.resolution}</span>
                                    <span className="size">{formatSize(image.size)}</span>
                                </div>
                            </div>
                        ))}
                    </div>

                    {/* 缩放指示器 */}
                    <div className={`zoom-indicator ${showZoomIndicator ? 'visible' : ''}`}>
                        {indicatorText}
                    </div>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
```

https://github.com/copilot/c/eb053817-aec3-4c4d-b3f4-252f1f13737b

