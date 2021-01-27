---
title: Unity拖动2D和3D物体
date:
updated:
tags: Unity技术
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover:
aplayer:
---
<meta name="referrer" content="no-referrer" />

  /*  
 首先说一下，Input.mousePosition是鼠标所在像素平面内的坐标，需要根据自己的需求转变成世界坐标。  
 Description 描述  
 The current mouse position in pixel coordinates. (Read Only)  

 在屏幕坐标空间当前鼠标的位置（只读）。  

 The bottom-left of the screen or window is at (0, 0). The top-right of the screen or window is at (Screen.width, Screen.height).  

 屏幕或窗口的左下角是坐标系的（0,0）坐标。右上角的坐标是（屏幕宽度值，屏幕高度值）。  

 */

```c#
    //这里用于2D场景物体的移动，可以参考一下。
    private bool isMouseDown;
    private Vector3 lastMousePosition = Vector3.zero;
    private void TwoDMove()
    {
        if (Input.GetMouseButtonDown(0))
        {
            isMouseDown = true;
        }
        if (Input.GetMouseButtonUp(0))
        {
            isMouseDown = false;
            lastMousePosition = Vector3.zero;//这里要归零，不然会有漂移效果
        }
        if (isMouseDown)
        {
            if (lastMousePosition != Vector3.zero)
            {
                Vector3 offset = Camera.main.ScreenToWorldPoint(Input.mousePosition) - lastMousePosition;
                transform.position += offset;
            }
            lastMousePosition = Camera.main.ScreenToWorldPoint(Input.mousePosition);


        }
    }

```



```c#
  //这里是使用Ray射线来控制物体移动，可是由于射线本身检测速率的限制，并不适合持续的跟踪移动。具体效果各位读者试试便知。
    private void RayMove()
    {
        Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        RaycastHit hit;
        if (isMouseDown)
        {
            if (Physics.Raycast(ray, out hit))
            {
                Vector3 offset = Input.mousePosition;
                hit.transform.position = new Vector3(hit.point.x, hit.point.y, hit.transform.position.z);
                Debug.DrawLine(ray.origin, hit.point);
            }
        }


    }
```



```c#
  /// <summary>
    /// 这里是3D物体的移动，注意坐标系的转换即可。
    /// </summary>
    /// <returns></returns>


    //注意世界坐标系转化为屏幕坐标系，Z轴是不变的
    IEnumerator OnMouseDown()
        {
            //将物体由世界坐标系转化为屏幕坐标系，由vector3 结构体变量ScreenSpace存储，以用来明确屏幕坐标系Z轴的位置
            Vector3 ScreenSpace = Camera.main.WorldToScreenPoint(transform.position);
            //完成了两个步骤，1由于鼠标的坐标系是2维的，需要转化成3维的世界坐标系，2只有三维的情况下才能来计算鼠标位置与物体的距离，offset即是距离
            Vector3 offset = transform.position - Camera.main.ScreenToWorldPoint(new Vector3(Input.mousePosition.x, Input.mousePosition.y, ScreenSpace.z));
            Debug.Log("down");
            //当鼠标左键按下时
            while (Input.GetMouseButton(0))
            {
                //得到现在鼠标的2维坐标系位置
                Vector3 curScreenSpace = new Vector3(Input.mousePosition.x, Input.mousePosition.y, ScreenSpace.z);
                //将当前鼠标的2维位置转化成三维的位置，再加上鼠标的移动量
                Vector3 CurPosition = Camera.main.ScreenToWorldPoint(curScreenSpace) + offset;
                //CurPosition就是物体应该的移动向量赋给transform的position属性
                transform.position = CurPosition;
                yield return new WaitForFixedUpdate();
        }
    }
```



 
