# 与飞行有关的`MoveControl`

本文把这类飞行生物使用的`MoveControl`称为飞行类`MoveControl`。

这一部分我们来看3个较泛用或简单的飞行类`MoveControl`的代码，来简单了解一下飞行类`MoveControl`的功能和组成部分，然后对比下这些飞行类`MoveControl`的区别。  

首先看比较泛用的`FlyingMoveControl`。

`FlyingMoveControl`中的`tick`方法：   
```java
if (operation == MoveControl.Operation.MOVE_TO) {
    operation = MoveControl.Operation.WAIT;
    mob.setNoGravity(true);
    double dx = wantedX - mob.getX();
    double dy = wantedY - mob.getY();
    double dz = wantedZ - mob.getZ();
    double distSqr = dx * dx + dy * dy + dz * dz;
    
    // 目标位置与当前位置相同
    if (distSqr < (double) 2.5000003E-7F) {
        mob.setYya(0);
        mob.setZza(0);
        return;
    }
    
    // 水平朝向（yRot） & 速度调整
    // 以90°的最大速度水平旋转向目标位置
    // 此处的反正切函数得到的度数是与x轴正方向（东）的夹角，而我们需要与z轴正方向（南）的夹角，所以要减去90°
    // 
    // 因为-atan2(a, b) = atan2(b, a) - 90°，所以也可以写成下面的语句
    // float newYRot = (float) -(Mth.atan2(dx, dz) * 180 / Math.PI);
    float newYRot = (float) (Mth.atan2(dz, dx) * 180 / Math.PI - 90);
    mob.setYRot(rotlerp(mob.getYRot(), newYRot, 90));
    float speed;
    if (mob.onGround()) {
        speed = (float) (speedModifier * mob.getAttributeValue(Attributes.MOVEMENT_SPEED));
    } else {
        speed = (float) (speedModifier * mob.getAttributeValue(Attributes.FLYING_SPEED));
    }
    mob.setSpeed(speed);
    
    // 垂直朝向（xRot） & 速度调整
    double horizontalDist = Math.sqrt(dx * dx + dz * dz);
    if (Math.abs(dy) > (double) 1.0E-5F || Math.abs(horizontalDist) > (double) 1.0E-5F) {
        // 注意“抬头”时xRot为负
        float newXRot = (float) -(Mth.atan2(dy, horizontalDist) * (180 / Math.PI));
        mob.setXRot(rotlerp(mob.getXRot(), newXRot, maxTurn));
        mob.setYya(dy > 0 ? speed : -speed);
    }
} else {
    // 自然降落
    if (!hoversInPlace) {
        mob.setNoGravity(false);
    }
    mob.setYya(0);
    mob.setZza(0);
}
```

以下是该`MoveControl`的核心特点：  
- 同时调整水平和垂直朝向
- 不飞行时自然降落
- 飞行速度与实体属性的值有关
  
接下来是`VexMoveControl`中的`tick`方法，`VexMoveControl`是恼鬼特有的`MoveControl`：  

```java
if (operation == MoveControl.Operation.MOVE_TO) {
    Vec3 vecToWanted = new Vec3(wantedX - getX(), wantedY - getY(), wantedZ - getZ());
    double len = vecToWanted.length();
    if (len < getBoundingBox().getSize()) {
        // 缓缓停下
        operation = MoveControl.Operation.WAIT;
        setDeltaMovement(getDeltaMovement().scale(0.5));
    } else {
        setDeltaMovement(getDeltaMovement().add(vecToWanted.scale(speedModifier * 0.05 / len)));
        
        if (getTarget() == null) {
            Vec3 deltaMovement = getDeltaMovement();
            setYRot((float) (-Mth.atan2(deltaMovement.x, deltaMovement.z) * 180 / Math.PI));
            yBodyRot = getYRot();
        } else {
            double dx = getTarget().getX() - getX();
            double dz = getTarget().getZ() - getZ();
            // 与FlyingMoveControl中不同，此处的反正切函数得到的度数是z轴正方向（南）的夹角，但是还要取相反数
            // 举个例子，当dx和dz都大于0时，应该面向东南方向，若原本面向正南（yRot=0），此时应逆时针旋转
            // 上方的if中根据移动方向自我调整朝向的代码同理
            setYRot((float) (-Mth.atan2(dx, dz) * 180 / Math.PI));
            yBodyRot = getYRot();
        }
    }
}
```

以下是该`MoveControl`的核心特点：  
- 只调整水平朝向
- 不飞行时不降落
- 飞行速度与实体属性的值无关  
  
不难发现，不同的点在于恼鬼始终保持着漂浮状态而不会自然下落，而且只会根据移动方向调整自身的水平朝向。  

然后是`GhastMoveControl`中的`tick`方法，`GhastMoveControl`是恶魂特有的`MoveControl`：  
```java
if (operation == MoveControl.Operation.MOVE_TO) {
    if (floatDuration-- <= 0) {
        floatDuration += ghast.getRandom().nextInt(5) + 2;
        Vec3 vecToWanted = new Vec3(wantedX - ghast.getX(), wantedY - ghast.getY(), wantedZ - ghast.getZ());
        double len  = vecToWanted.length();
        vecToWanted = vecToWanted.normalize();
        if (canReach(vecToWanted, Mth.ceil(len))) {
            ghast.setDeltaMovement(ghast.getDeltaMovement().add(vecToWanted.scale(0.1)));
        } else {
            operation = MoveControl.Operation.WAIT;
        }
    }
}
```

以下是该`MoveControl`的核心特点：  
- 不调整朝向（恶魂的朝向在其AI中调整而不在此处调整）
- 不飞行时不降落
- 飞行速度与实体属性的值无关  

恶魂的移动则简单很多，只会间歇性漂移。其朝向和是否正在看向其攻击目标有关，而且要分类讨论，这也许是代码中把恶魂的朝向用单独的`GhastLookGoal`来控制的原因。  

通过对比这3个`MoveControl`，我们也可以得知缓慢效果影响不了飞行速度，而且很难统一改变所有飞行生物的飞行速度的原因。~~原版MC的代码还是太难以评价了。~~  

据此可以总结出`MoveControl`一般会具备的功能：
1. 改变实体的移动方向（通过改变`deltaMovement`实现）
2. 改变实体的朝向，包括水平朝向（`yRot`）和垂直朝向（`xRot`）  

飞行的友好生物（鹦鹉、蜜蜂、悦灵等）一般使用`FlyingMoveControl`，而飞行的怪物移动方式较为复杂，因此一般使用自定义的飞行类`MoveControl`。