#### unreal使用的坐标系

1. unreal使用的坐标系是左手坐标系，与DirectX相同，与OpenGL相反

2. x正方向为向前，y正方向向左，z正方向向上

3. 左手坐标系转换为右手坐标系的方法，z取负即可，比如 左手坐标系坐标为```(x,y,z)```，转换为右手坐标系的坐标则为```(x,y,-z)```

4. 编辑器中的location, rotation，有Relative和World可以选择，World很好理解，就是世界坐标系下的，而Relative是指相对于选定的object的父object的location和rotation，如果选定的object没有父object，那么relative和world是没有区别的。

5. UV坐标，U是横坐标，V是纵坐标，原点位于左下角。大小为[0，1]